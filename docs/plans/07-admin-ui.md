# 07 — Admin UI & LangGraph 시각화

## 개요

Firebase Hosting으로 제공되는 단일 페이지 앱(SPA).
Firestore 실시간 리스닝(onSnapshot)으로 에이전트 실행 상태를 실시간 반영.
LangGraph 노드의 점멸, 실행 로그, 전략 조작을 하나의 화면에서 제공한다.

---

## 1. 화면 구성

```
┌─────────────────────────────────────────────────────────────┐
│  Stock Manager Admin         [페이퍼 모드] [비상 정지] [로그아웃]│
├──────────┬──────────────────────────────┬───────────────────┤
│ 사이드바  │     LangGraph 실행 그래프     │   실행 로그        │
│          │                              │                   │
│ ▶ 그래프  │   [ScannerAgent] ────────►  │ 14:30:01 SCAN    │
│   대시보드│         │                   │ 14:30:02 FILTER  │
│   포지션  │         ▼                   │ 14:30:05 PATTERN │
│   전략    │   [FilterAgent]  ────────►  │   AAPL: golden..│
│   리포트  │         │                   │ 14:30:08 SIGNAL  │
│   설정    │         ▼                   │   BUY AAPL @175 │
│          │   [PatternAgent] ────────►  │ 14:30:10 RISK   │
│          │         │                   │   pos size: 5%  │
│          │   [SignalAgent]             │ ...              │
│          │         │                   │                   │
│          │      [fork]                 │                   │
│          │    ↙       ↘              │                   │
│          │[RiskAgent][MonitorAgent]   │                   │
│          │    ↘       ↙              │                   │
│          │  [PriorityAgent]            │                   │
│          │         │                   │                   │
│          │  [ExecutionAgent]           │                   │
│          │                              │                   │
├──────────┴──────────────────────────────┴───────────────────┤
│  포트폴리오 요약: 총 평가액 10,125,000원 (+1.25%) | 보유 3종목 │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. LangGraph 그래프 시각화 (핵심 기능)

### 기술 구현

```
라이브러리: React Flow (https://reactflow.dev)
  - 노드: Agent별 커스텀 노드 컴포넌트
  - 엣지: 조건부 엣지 (레이블 포함)
  - 레이아웃: dagre 라이브러리로 자동 계층 배치
```

### 노드 상태 표현

```typescript
type NodeStatus = "idle" | "running" | "success" | "error" | "skipped";

// Firestore /agent_runs/{runId}/node_logs에서 실시간 읽기
interface NodeLog {
  node_name: string;
  status: NodeStatus;
  started_at: Timestamp | null;
  finished_at: Timestamp | null;
  result_summary: string;      // 짧은 결과 요약
  error: string | null;
}
```

### 노드 시각적 표현

| 상태 | 색상 | 애니메이션 |
|------|------|-----------|
| idle | 회색 | 없음 |
| running | 파란색 | 점멸 (pulse) |
| success | 초록색 | 0.5초 강조 후 유지 |
| error | 빨간색 | 흔들림 (shake) |
| skipped | 연회색 | 점선 테두리 |

### 노드 클릭 시 상세 패널

```
[PatternAgent] 클릭 →
┌──────────────────────────────┐
│ PatternAgent 상세             │
│ 상태: ✅ success              │
│ 실행 시간: 2.3초              │
│ 처리 종목: 24개               │
│ 패턴 감지:                    │
│   AAPL: golden_cross (0.85)  │
│   MSFT: bull_flag (0.72)     │
│   TSLA: rsi_divergence (0.68)│
└──────────────────────────────┘
```

### 실시간 업데이트 구현

```typescript
// Firestore 실시간 리스닝
import { onSnapshot, doc } from "firebase/firestore";

function useAgentRun(runId: string) {
  const [nodeStatuses, setNodeStatuses] = useState<Record<string, NodeLog>>({});

  useEffect(() => {
    const unsubscribe = onSnapshot(
      doc(db, "agent_runs", runId),
      (snapshot) => {
        const data = snapshot.data();
        setNodeStatuses(data?.node_logs ?? {});
      }
    );
    return () => unsubscribe();
  }, [runId]);

  return nodeStatuses;
}
```

### Agent 실행 시 Firestore 업데이트 (Python 측)

```python
def run_node_with_logging(node_fn, state: GraphState, node_name: str):
    """노드 실행을 감싸서 Firestore에 상태 기록"""
    run_ref = db.collection("agent_runs").document(state["run_id"])

    # 실행 시작 기록
    run_ref.update({
        f"node_logs.{node_name}": {
            "status": "running",
            "started_at": firestore.SERVER_TIMESTAMP,
            "finished_at": None,
            "result_summary": "",
            "error": None,
        }
    })

    try:
        result = node_fn(state)
        # 성공 기록
        run_ref.update({
            f"node_logs.{node_name}.status": "success",
            f"node_logs.{node_name}.finished_at": firestore.SERVER_TIMESTAMP,
            f"node_logs.{node_name}.result_summary": summarize_result(result),
        })
        return result

    except Exception as e:
        # 오류 기록
        run_ref.update({
            f"node_logs.{node_name}.status": "error",
            f"node_logs.{node_name}.finished_at": firestore.SERVER_TIMESTAMP,
            f"node_logs.{node_name}.error": str(e),
        })
        raise
```

---

## 3. 주요 화면별 기능

### 3.1 그래프 화면 (기본 홈)

- LangGraph 실행 그래프 실시간 표시
- 최근 N개 실행 선택 탭
- "현재 실행 중" 자동 포커스
- 우측 패널: 실시간 스트리밍 로그

### 3.2 대시보드

```
┌──────────┬──────────┬──────────┬──────────┐
│ 총 평가액 │ 오늘 수익│ 월 누적  │ 승률     │
│ 10.1M원  │ +125K   │ +1.25%  │ 68%      │
└──────────┴──────────┴──────────┴──────────┘

포트폴리오 가치 추이 (선 차트)
종목별 수익률 바 차트
전략별 성과 파이 차트
```

### 3.3 포지션 화면

```
보유 포지션 테이블:
종목 | 수량 | 평균단가 | 현재가 | 손익 | 손절가 | 목표가 | 전략 | [청산]

청산 버튼 → POST /api/positions/{id}/close 호출
```

### 3.4 전략 화면 ([05-strategy-priority.md] 참조)

- 전략별 가중치 슬라이더
- 활성화/비활성화 토글
- 최근 성과 지표
- 가중치 변경 이력 차트

### 3.5 리포트 화면

- 일별/주별/월별 성과 리포트
- 전략별 상세 분석
- 패턴별 성공률 표

### 3.6 설정 화면

```
시스템 설정:
  - 거래 모드 (페이퍼 / 라이브)
  - 초기 자금 설정 (페이퍼)
  - 최대 포지션 수
  - 일일 최대 거래 횟수

종목 우주 관리:
  - 관심 종목 추가/제거
  - 블랙리스트 관리

알림 설정:
  - 이메일 알림
  - Slack Webhook URL
```

---

## 4. 인증

```
Firebase Authentication:
  - Google 로그인 (주요)
  - 허용 이메일 목록: config.yaml에 정의

Firestore 보안 규칙:
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null
        && request.auth.token.email in [
          'admin@example.com'
        ];
    }
  }
}
```

---

## 5. 배포 구성

```bash
# Firebase Hosting 배포
firebase deploy --only hosting

# Firebase Functions 배포
firebase deploy --only functions

# Cloud Run Job 배포
gcloud run jobs deploy agent-runner \
  --image gcr.io/PROJECT_ID/agent-runner:latest \
  --region asia-northeast3 \
  --task-timeout 600s \
  --max-retries 2

# Cloud Scheduler 등록
gcloud scheduler jobs create http intraday-scan \
  --schedule "*/15 9-15 * * 1-5" \
  --time-zone "Asia/Seoul" \
  --uri "https://REGION-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/PROJECT_ID/jobs/agent-runner:run" \
  --http-method POST \
  --oauth-service-account-email SCHEDULER_SA@PROJECT_ID.iam.gserviceaccount.com
```

---

## 6. 로컬 개발 환경

```bash
# Firebase 에뮬레이터 (Firestore, Functions, Hosting)
firebase emulators:start

# Admin UI 개발 서버
cd admin-ui && npm run dev

# Agent Runner 로컬 실행
cd agent-runner
python main.py --mode paper --schedule intraday-scan

# 환경 변수
FIRESTORE_EMULATOR_HOST=localhost:8080
GOOGLE_APPLICATION_CREDENTIALS=./service-account.json
```

### LangGraph Studio 통합 (선택적)

로컬 개발 시 LangGraph Studio (`langgraph studio`)를 사용하면
표준 LangGraph UI로 그래프를 시각화하고 디버깅할 수 있다.
프로덕션에서는 커스텀 Admin UI를 사용한다.
