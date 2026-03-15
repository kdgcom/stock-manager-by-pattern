# 04 — 스케쥴링 전략

## 개요

Cloud Scheduler + Cloud Run Jobs 조합으로 구현.
시장 시간대별로 다른 스캔 전략을 적용한다.

---

## 1. 스케쥴 구성

### 미국 주식 기준 (EST)

| 스케쥴 이름 | Cron | 작업 | 비고 |
|------------|------|------|------|
| pre-market-scan | `30 8 * * 1-5` | 프리마켓 스캔 | 장 전 준비 |
| market-open-scan | `35 9 * * 1-5` | 개장 직후 스캔 | 갭업/갭다운 |
| intraday-scan | `*/15 10-15 * * 1-5` | 장중 정기 스캔 | 15분 간격 |
| market-close-scan | `55 15 * * 1-5` | 마감 전 스캔 | 청산 여부 확인 |
| post-market-analysis | `0 17 * * 1-5` | 당일 결과 분석 | 전략 성과 기록 |
| weekly-calibration | `0 20 * * 5` | 전략 가중치 재조정 | 주간 성과 반영 |

### 한국 주식 기준 (KST)

| 스케쥴 이름 | Cron (KST) | 작업 |
|------------|------------|------|
| pre-market-scan | `0 9 * * 1-5` | 장 전 스캔 |
| market-open-scan | `5 9 * * 1-5` | 개장 후 스캔 |
| intraday-scan | `*/15 9-15 * * 1-5` | 장중 15분 스캔 |
| market-close-scan | `20 15 * * 1-5` | 마감 전 청산 확인 |

> 시장 설정은 `config.yaml`의 `market.timezone`으로 결정.

---

## 2. 스케쥴별 작업 상세

### pre-market-scan (장 전)

```
목적: 당일 관심 종목 선정
LangGraph 노드 실행:
  ScannerAgent → FilterAgent → PatternAgent (일봉 기준)

특징:
  - 일봉(1D) OHLCV 데이터 기준
  - 전일 종가 패턴 집중 분석
  - 결과를 Firestore /scan_results에 저장 (장중 스캔의 기준)
  - MonitorAgent: 전일 보유 포지션 갭 리스크 점검
```

### market-open-scan (개장 직후)

```
목적: 갭 트레이딩 기회 포착
LangGraph 노드 실행:
  ScannerAgent (갭 필터 강화) → FilterAgent → SignalAgent → RiskAgent → ExecutionAgent

특징:
  - 전일 종가 대비 갭 비율 우선 필터
  - 빠른 실행 우선 (전체 패턴 분석보다 갭+거래량 신호)
  - 타임아웃: 5분 이내
```

### intraday-scan (장중 15분)

```
목적: 장중 패턴 포착 및 포지션 모니터링
LangGraph 노드 실행:
  ScannerAgent → FilterAgent → PatternAgent (분봉 기준) → SignalAgent
  → RiskAgent → PriorityAgent → ExecutionAgent + MonitorAgent (병렬)

특징:
  - 15분봉 / 1시간봉 데이터 혼합 사용
  - pre-market-scan 결과를 필터의 기준으로 활용
  - MonitorAgent: 손절/목표가 도달 여부 실시간 체크
```

### market-close-scan (마감 전)

```
목적: 당일 청산 결정
LangGraph 노드 실행:
  MonitorAgent → ExecutionAgent

특징:
  - 신규 진입 없음
  - `close_all_intraday: true` 설정 시 당일 포지션 전량 청산
  - 스윙 포지션은 유지
```

### post-market-analysis (장 후)

```
목적: 당일 거래 성과 분석 및 기록
LangGraph 노드 실행:
  AnalysisAgent (별도 노드, 정규 그래프 외)

작업:
  - 당일 실행된 전략별 수익률 계산
  - 패턴별 성공/실패 기록
  - Firestore /strategies의 performance_score 업데이트
  - Admin UI 대시보드용 일일 리포트 생성
```

### weekly-calibration (주간)

```
목적: 전략 가중치 자동 재조정
알고리즘:
  1. 최근 4주 전략별 성과 (승률, 수익률, 샤프지수) 계산
  2. 성과 점수 정규화
  3. 새 가중치 = 현재 가중치 × (1 - learning_rate) + 성과 점수 × learning_rate
  4. 가중치 범위 클리핑 [min_weight, max_weight]
  5. Firestore /strategies 업데이트
  6. 사용자 알림 (이메일 또는 Admin UI 알림)
```

---

## 3. 종목 우주(Universe) 스캔 전략

### 우주 정의

```
전체 유니버스 계층 구조:

Level 1 (항상 스캔): watchlist
  - 사용자가 명시적으로 추가한 종목
  - 최대 50개 권장

Level 2 (주기적 스캔): core_universe
  - 시총 상위 N개 (기본 S&P 500 구성 종목)
  - 섹터별 대표 ETF 포함

Level 3 (확장 스캔): extended_universe
  - Level 2 + 중소형주
  - 거래량 상위 기준 동적 선정
  - 주간 1회 업데이트
```

### 스캔 분할 전략

Firestore 읽기 제한(50,000/일)을 고려한 분할:

```
장 전 스캔:
  - Level 1 (watchlist): 전수 스캔
  - Level 2 (core): 전수 스캔
  - 총 읽기: ~550 종목 × 100봉 = 55,000행
  → Firestore: 1회 읽기로 배치 로드 (종목 메타) + 브로커 API로 OHLCV

장중 스캔 (15분):
  - Level 1: 항상
  - Level 2: 거래량 상위 30% 만 (필터링)
  - Level 3: 랜덤 샘플링 (10%)
  → 총 Firestore 읽기 최소화
```

### 동적 우주 업데이트

```
트리거: weekly-calibration 실행 시
작업:
  1. 브로커 API로 최근 5거래일 거래량 상위 200개 조회
  2. 기존 extended_universe와 병합
  3. 거래량 하위 10% 제거
  4. 새 종목 추가 시 최소 20일 히스토리 데이터 백필
```

---

## 4. 타임아웃 및 오류 처리

### Cloud Run Job 타임아웃 설정

```yaml
job_timeout:
  pre_market_scan: 300s      # 5분
  market_open_scan: 180s     # 3분
  intraday_scan: 240s        # 4분
  market_close_scan: 120s    # 2분
  post_market_analysis: 600s # 10분
  weekly_calibration: 900s   # 15분
```

### 실패 처리

```
재시도 정책:
  - 최대 재시도: 2회
  - 재시도 간격: 30초
  - 재시도 시 이전 상태 없음 (멱등성 보장)

부분 실패:
  - Agent 노드별 오류는 node_logs에 기록 후 다음 노드 계속
  - ExecutionAgent 오류 → 주문 미실행, 알림 발송
  - 연속 3회 실패 시 전체 실행 일시 중단 + 알림

알림 채널 (설정 가능):
  - 이메일 (SendGrid)
  - Slack Webhook
  - Firebase Cloud Messaging (Admin UI)
```

---

## 5. 스케쥴 상태 모니터링

Firestore `/schedule_runs` 컬렉션:
```
{runId}:
  schedule_name: "intraday-scan"
  triggered_at: timestamp
  started_at: timestamp
  finished_at: timestamp
  status: "success" | "failed" | "timeout" | "skipped"
  nodes_executed: [...]
  orders_executed: 3
  errors: []
```

Admin UI에서 실시간 스케쥴 실행 현황 확인 가능.
