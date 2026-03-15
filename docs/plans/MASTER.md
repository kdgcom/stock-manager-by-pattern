# Stock Manager by Pattern — 마스터 설계 문서

## 프로젝트 개요

차트 패턴 인식 기반의 주식 자동 거래 Agent 시스템.
LangGraph 기반의 멀티-에이전트 아키텍처로 구성되며, GCP 위에서 동작한다.
알고리즘 기반 결정과 LLM 보조 결정을 유연하게 전환할 수 있다.

---

## 핵심 설계 결정

| 항목 | 결정 | 근거 |
|------|------|------|
| 실행 환경 | Cloud Run Jobs + Cloud Scheduler | Firebase Functions 무료 티어의 CPU-second 한계 초과 가능성, LangGraph 컨테이너 실행 필요 |
| DB | Firestore | 실시간 업데이트, 문서 기반 유연성, Firebase 생태계 |
| Admin UI | Firebase Hosting (SPA) | 무료 호스팅, Firestore 실시간 리스닝 |
| API 엔드포인트 | Firebase Functions (2nd gen) | 가벼운 HTTP 트리거, 무료 티어로 충분 |
| Agent 오케스트레이션 | LangGraph (Python) | 상태 기계 기반 그래프, 시각화, 조건부 엣지 |
| 데이터 소스 | 공개 API 우선 → 삼성증권 API 폴백 | 비용 절감, API 호출 한도 관리 |
| 브로커 | 삼성증권 API | 매매 실행, 공개 API 불가 시 시세 폴백 |
| LLM | 설정 파일로 선택 (OpenAI / Anthropic / 없음) | 유연성, 비용 제어 |
| 시뮬레이션 | 페이퍼 트레이딩 모드 (Firestore 가상 계좌) | 실제 주문 전 검증, 다중 안전장치 |

---

## 문서 구조

```
docs/plans/
├── MASTER.md                    # 이 파일 (전체 개요, 결정, 연결)
├── 01-architecture.md           # 시스템 아키텍처, 인프라 구성
├── 02-agents.md                 # Agent 정의 및 결정 로직
├── 03-patterns.md               # 차트 패턴 인식 함수 목록
├── 04-scheduling.md             # 스케쥴링 전략 및 스캔 흐름
├── 05-strategy-priority.md      # 전략 우선순위 시스템
├── 06-simulation.md             # 페이퍼 트레이딩 시뮬레이션
├── 07-admin-ui.md               # LangGraph 시각화 Admin UI
└── 08-config.md                 # 설정 파일 스키마 및 참조
```

---

## 전체 시스템 흐름 (요약)

```
Cloud Scheduler
    │
    ▼ (Cron trigger)
Cloud Run Job: agent-runner
    │
    ▼
LangGraph Graph 실행
    ├── ScannerAgent          : 종목 우주(universe) 스캔
    │                           (공개 API 우선, 삼성증권 API 폴백)
    ├── FilterAgent           : 기초 조건 필터링
    ├── PatternAgent          : 차트 패턴 인식
    ├── SignalAgent           : 매매 신호 생성
    ├── RiskAgent             : 리스크 평가 & 포지션 사이징
    ├── PriorityAgent         : 전략 우선순위 적용
    ├── ExecutionAgent        : 주문 실행 (또는 시뮬레이션)
    │                           (paper: Firestore 기록만 / live: 삼성증권 API)
    └── MonitorAgent          : 보유 포지션 모니터링
    │
    ▼
Firestore (상태, 로그, 포지션, 전략 가중치)
    │
    ▼
Firebase Hosting Admin UI (실시간 그래프 시각화)
```

---

## 개발 단계 (Phase)

### Phase 1 — 기반 구축
- Cloud Run + Firestore 연결
- 설정 파일 로딩 구조
- 페이퍼 트레이딩 계좌 모델
- 주요 패턴 인식 함수 5종 구현

### Phase 2 — Agent 오케스트레이션
- LangGraph 그래프 정의
- 각 Agent 알고리즘 구현
- 스케쥴링 연결

### Phase 3 — 우선순위 & 사용자 개입
- 전략 가중치 시스템
- Admin API (Firebase Functions)
- Admin UI MVP

### Phase 4 — LLM 통합 & 시각화
- LLM 보조 결정 레이어
- LangGraph 실시간 시각화 (Admin UI)
- 성능 기반 자동 가중치 조정

---

## 참조 문서

- [01-architecture.md](./01-architecture.md) — 인프라, 기술 스택 상세
- [02-agents.md](./02-agents.md) — Agent별 결정 로직
- [03-patterns.md](./03-patterns.md) — 패턴 인식 함수 목록
- [04-scheduling.md](./04-scheduling.md) — 스캔 스케쥴 전략
- [05-strategy-priority.md](./05-strategy-priority.md) — 전략 우선순위 관리
- [06-simulation.md](./06-simulation.md) — 페이퍼 트레이딩
- [07-admin-ui.md](./07-admin-ui.md) — Admin UI 설계
- [08-config.md](./08-config.md) — 설정 파일 스키마
