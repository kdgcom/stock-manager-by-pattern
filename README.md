# Stock Manager by Pattern

차트 패턴 인식 기반의 주식 자동 거래 Agent 시스템.
LangGraph 멀티-에이전트 아키텍처로 GCP 위에서 동작하며,
알고리즘 기반 결정과 LLM 보조 결정을 유연하게 전환할 수 있다.

---

## 주요 특징

- **패턴 인식**: 20+ 차트 패턴을 개별 함수로 구현 (헤드앤숄더, 골든크로스, 볼린저 돌파 등)
- **멀티-에이전트**: LangGraph 기반 8개 Agent가 협력하여 스캔 → 분석 → 실행 수행
- **페이퍼 트레이딩**: 가상 계좌로 전략 검증 후 라이브 전환
- **실시간 시각화**: Admin UI에서 LangGraph 노드 점멸 및 실행 로그 실시간 확인
- **전략 우선순위**: 성과 기반 자동 가중치 + 사용자 직접 개입 지원
- **LLM 선택적 사용**: 설정으로 OpenAI/Anthropic/Custom LLM 연결 (없어도 동작)

---

## 아키텍처 결정

| 컴포넌트 | 기술 | 이유 |
|----------|------|------|
| Agent 실행 | Cloud Run Jobs | LangGraph 컨테이너, 장시간 실행 지원 |
| 스케쥴링 | Cloud Scheduler | Cron 기반, Cloud Run Job 트리거 |
| DB | Firestore | 실시간 리스닝, 무료 티어 |
| 데이터 소스 | 공개 API 우선 (yfinance, KRX) | 무료, 별도 인증 불필요 |
| 브로커 | 삼성증권 API | 매매 실행 + 공개 API 불가 시 시세 폴백 |
| API | Firebase Functions | 가벼운 HTTP 트리거 |
| Admin UI | Firebase Hosting | SPA, React Flow 그래프 시각화 |
| 언어 | Python 3.11+ | LangGraph, pandas-ta, pydantic |

> **데이터 소스 우선순위**: 시세 데이터는 공개 API(yfinance, KRX, 네이버 금융)를 먼저 사용하고,
> 분봉 등 공개 API가 제공하지 않는 데이터는 삼성증권 API로 폴백한다.
> **시뮬레이션 모드**에서는 브로커 주문 API에 절대 연결하지 않으며,
> Firestore 가상 계좌에만 기록한다. 라이브 전환 시 3중 확인(config + 환경변수 + Firestore)을 요구한다.

---

## Agent 구성

```
ScannerAgent   → FilterAgent → PatternAgent → SignalAgent
                                                    │
                                        ┌───────────┴───────────┐
                                        ▼                       ▼
                                   RiskAgent             MonitorAgent
                                        │                       │
                                        └───────────┬───────────┘
                                                    ▼
                                            PriorityAgent
                                                    │
                                            ExecutionAgent
```

---

## 빠른 시작

### 1. 사전 요구 사항

- GCP 프로젝트 생성, Firebase 연결
- Python 3.11+, Node.js 18+
- Firebase CLI, gcloud CLI

### 2. 설정 파일 준비

```bash
cp config/config.example.yaml config/config.yaml
# config.yaml 편집 (브로커 API 키, 거래 모드 등)
```

### 3. 로컬 개발 실행

```bash
# Firebase 에뮬레이터 시작
firebase emulators:start

# Admin UI 개발 서버
cd admin-ui && npm install && npm run dev

# Agent Runner 로컬 실행 (페이퍼 트레이딩)
cd agent-runner
pip install -r requirements.txt
python main.py --mode paper --schedule intraday-scan
```

### 4. GCP 배포

```bash
# Cloud Run Job 빌드 및 배포
cd agent-runner
gcloud builds submit --tag gcr.io/PROJECT_ID/agent-runner
gcloud run jobs deploy agent-runner \
  --image gcr.io/PROJECT_ID/agent-runner \
  --region asia-northeast3

# Firebase 배포 (Functions + Hosting)
firebase deploy
```

---

## 전략 우선순위 조작

Admin UI (`https://YOUR_PROJECT.web.app`) 접속 → **전략** 탭:
- 전략별 가중치 슬라이더 조작
- 활성화/비활성화 토글
- 비상 정지 버튼

REST API로도 조작 가능:
```bash
# 특정 전략 가중치 변경
curl -X PATCH https://REGION-PROJECT_ID.cloudfunctions.net/api/strategies/trend_follow \
  -H "Authorization: Bearer TOKEN" \
  -d '{"user_weight_override": 0.9}'

# 종목 블랙리스트 추가
curl -X POST .../api/blacklist \
  -d '{"ticker": "TSLA", "reason": "실적 발표 대기"}'
```

---

## 설계 문서

전체 설계 내용은 [`docs/plans/`](docs/plans/) 디렉토리에 있다.

| 문서 | 내용 |
|------|------|
| [MASTER.md](docs/plans/MASTER.md) | 전체 개요, 주요 결정, 문서 인덱스 |
| [01-architecture.md](docs/plans/01-architecture.md) | 인프라, 기술 스택, Firestore 스키마 |
| [02-agents.md](docs/plans/02-agents.md) | Agent별 결정 로직 및 LangGraph 그래프 |
| [03-patterns.md](docs/plans/03-patterns.md) | 20+ 차트 패턴 인식 함수 명세 |
| [04-scheduling.md](docs/plans/04-scheduling.md) | 스케쥴 구성, 종목 우주 전략 |
| [05-strategy-priority.md](docs/plans/05-strategy-priority.md) | 전략 가중치 시스템, 사용자 개입 방법 |
| [06-simulation.md](docs/plans/06-simulation.md) | 페이퍼 트레이딩, 슬리피지 모델 |
| [07-admin-ui.md](docs/plans/07-admin-ui.md) | LangGraph 시각화 Admin UI 설계 |
| [08-config.md](docs/plans/08-config.md) | 설정 파일 전체 스키마 |

---

## 라이선스

Private — 무단 배포 금지
