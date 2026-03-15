# 01 — 시스템 아키텍처

## 1. 인프라 구성

### Firebase Functions vs Cloud Run 결정

Firebase Functions 무료 티어 한계:
- 호출 횟수: 125,000/월 (2nd gen: 2,000,000/월)
- CPU-seconds: 40,000/월 (2nd gen: 180,000/월)
- 최대 실행 시간: 60분 (2nd gen)
- Cold start 지연: LangGraph 컨테이너에 불리

**결론: 주 에이전트 루프는 Cloud Run Jobs 사용.**

| 컴포넌트 | 서비스 | 이유 |
|----------|--------|------|
| 에이전트 실행 루프 | Cloud Run Jobs | 컨테이너 기반, 장시간 실행, LangGraph 지원 |
| 스케쥴링 | Cloud Scheduler | Cron 표현식, Cloud Run Job 트리거 |
| API 엔드포인트 | Firebase Functions (2nd gen) | 가벼운 HTTP, 무료 티어 충분 |
| 데이터베이스 | Firestore | 실시간 리스닝, 무료 티어 50K reads/day |
| Admin UI 호스팅 | Firebase Hosting | 무료, CDN |
| 시크릿 관리 | GCP Secret Manager | API 키, 브로커 인증 정보 |

### 무료 티어 체크리스트

```
Cloud Run Jobs:
  - 180,000 vCPU-seconds/월 free
  - 스캔 1회 = ~30초 CPU → 월 6,000회 실행 가능
  - 실제 거래일: 22일 × 48회/일 = 1,056회 → 여유 있음

Firestore:
  - 읽기 50,000/일, 쓰기 20,000/일
  - 스캔 1회당 읽기 ~500건 → 50K/500 = 100회/일 한계
  - 최적화 필요: 배치 읽기, 캐싱 전략

Firebase Hosting:
  - 1GB 저장소, 10GB/월 전송 → Admin UI에 충분

Firebase Functions (API 용도):
  - 2M 호출/월 → API 엔드포인트에 충분
```

---

## 2. 기술 스택

### 백엔드 (Cloud Run Job)

```
언어:     Python 3.11+
프레임워크:
  - LangGraph          : 에이전트 오케스트레이션
  - langgraph-sdk      : 상태 직렬화
  - pandas / numpy     : 시계열 데이터 처리
  - ta-lib (또는 pandas-ta) : 기술적 지표 계산
  - firebase-admin     : Firestore 연결
  - google-cloud-secret-manager : 시크릿 로딩
  - httpx              : 브로커 API / LLM API 호출
  - pydantic v2        : 설정 및 데이터 모델 검증
```

### 데이터 소스 계층 (공개 API 우선 원칙)

```
시세/차트 데이터 조회 시 우선순위:

1순위: 공개 API (무료, API 키 불필요 또는 무료 키)
  - KRX 정보데이터시스템   : 한국 종목 일봉, 종목 정보
  - Yahoo Finance (yfinance): 글로벌 종목 OHLCV, 기본적 지표
  - FRED API              : 금리, 환율 등 매크로 지표
  - Naver Finance 크롤링   : 한국 종목 보조 데이터

2순위: 삼성증권 API (인증 필요)
  - 공개 API 미지원 데이터 (분봉, 실시간 호가 등)
  - 실제 매매 주문 실행
  - 계좌 잔고 및 포지션 조회

인터페이스: DataSourceRouter
  → 데이터 유형별로 1순위 시도 → 실패 시 2순위 폴백
```

### 브로커 연동 (추상화 레이어)

```
인터페이스: BrokerAdapter (추상 클래스)
구현체:
  - SamsungBrokerAdapter : 삼성증권 API (매매 + 폴백 시세)
  - PaperBrokerAdapter   : 페이퍼 트레이딩 (Firestore 기반)

인터페이스: QuoteProvider (추상 클래스, 시세 전용)
구현체:
  - PublicQuoteProvider   : 공개 API 통합 (yfinance, KRX 등)
  - SamsungQuoteProvider  : 삼성증권 API 시세 조회
  - FallbackQuoteProvider : PublicQuote → SamsungQuote 자동 폴백
```

### Admin UI (Firebase Hosting)

```
프레임워크: React + Vite (또는 Svelte)
상태 관리:  Firestore 실시간 리스닝 (onSnapshot)
그래프 시각화: React Flow (LangGraph 노드 렌더링)
차트:       Recharts 또는 lightweight-charts
```

---

## 3. 디렉토리 구조 (예정)

```
stock-manager-by-pattern/
├── agent-runner/                # Cloud Run Job 컨테이너
│   ├── Dockerfile
│   ├── main.py                  # 진입점
│   ├── graph/
│   │   ├── graph.py             # LangGraph 그래프 정의
│   │   ├── state.py             # GraphState 모델
│   │   └── nodes/               # 각 Agent 노드
│   │       ├── scanner.py
│   │       ├── filter.py
│   │       ├── pattern.py
│   │       ├── signal.py
│   │       ├── risk.py
│   │       ├── priority.py
│   │       ├── execution.py
│   │       └── monitor.py
│   ├── patterns/                # 패턴 인식 함수들
│   ├── brokers/                 # 브로커 어댑터
│   ├── datasources/             # 공개 API 연동 (yfinance, KRX 등)
│   ├── services/                # Firestore, Secret Manager
│   └── config.py                # 설정 로딩
│
├── functions/                   # Firebase Functions (API)
│   ├── src/
│   │   ├── strategy.ts          # 전략 가중치 CRUD
│   │   ├── portfolio.ts         # 포트폴리오 조회
│   │   └── admin.ts             # Admin 명령 수신
│   └── package.json
│
├── admin-ui/                    # Firebase Hosting
│   ├── src/
│   │   ├── components/
│   │   │   ├── GraphViewer.tsx  # LangGraph 시각화
│   │   │   ├── LogStream.tsx    # 실시간 로그
│   │   │   └── StrategyPanel.tsx# 전략 우선순위 조작
│   │   └── App.tsx
│   └── package.json
│
├── config/
│   ├── config.example.yaml      # 설정 예시 (커밋됨)
│   └── config.yaml              # 실제 설정 (gitignore)
│
└── docs/
    ├── first-order.md
    └── plans/
        └── (이 문서들)
```

---

## 4. Firestore 컬렉션 설계

```
/config
  └── {configId}                 # 시스템 설정 스냅샷

/strategies
  └── {strategyId}
        name, weight, enabled, performance_score, updated_at

/universe
  └── {ticker}
        ticker, exchange, sector, enabled, last_scanned_at

/scan_results
  └── {runId}
        timestamp, candidates[], patterns_found[], signals[]

/positions
  └── {positionId}
        ticker, mode (paper|live), entry_price, qty,
        strategy_id, opened_at, status (open|closed)

/orders
  └── {orderId}
        ticker, side (buy|sell), qty, price, mode,
        strategy_id, status, created_at, executed_at

/agent_runs
  └── {runId}
        started_at, finished_at, mode, nodes_executed[],
        node_logs {nodeName: {started, finished, result}}

/paper_account
  └── default
        cash, total_value, positions_value, updated_at
```

---

## 5. 데이터 흐름 다이어그램

```
[Cloud Scheduler] ──cron──► [Cloud Run Job]
                                    │
                                    ▼
                             [LangGraph 실행]
                             ┌──────────────┐
                             │  GraphState  │  (메모리 내 상태)
                             └──────────────┘
                                    │
              ┌──────────┬─────────┼──────────┬───────────┐
              ▼          ▼         ▼          ▼           ▼
        [Firestore] [공개 API] [삼성증권]  [LLM API]     │
        (상태영속화) (시세 1순위) (매매/폴백) (분석 보조)    │
              │                                           │
              ▼                                           │
       [Firebase Hosting]◄────────────────────────────────┘
       [Admin UI onSnapshot]

데이터 소스 우선순위:
  시세 조회: 공개 API → 삼성증권 API (폴백)
  매매 주문: 삼성증권 API (라이브) / Firestore (페이퍼)
```

---

## 6. 보안 고려 사항

- 브로커 API 키, LLM API 키 → GCP Secret Manager 저장
- Cloud Run Job 서비스 계정 → 최소 권한 원칙
- Firebase Functions API → Firebase Auth 또는 API Key 인증
- Admin UI → Firebase Auth (Google 로그인)
- Firestore 보안 규칙 → 인증된 관리자만 쓰기 허용
