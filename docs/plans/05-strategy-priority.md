# 05 — 전략 우선순위 시스템

## 개요

각 전략(패턴 + 매매 방식의 조합)은 가중치를 가지며,
이 가중치가 신호의 최종 실행 우선순위를 결정한다.
사용자는 언제든 이 가중치에 직접 개입할 수 있다.

---

## 1. 전략 정의

### 전략 = 패턴 그룹 + 시장 컨텍스트 + 매매 방식

```python
@dataclass
class Strategy:
    id: str                          # 고유 식별자
    name: str                        # 사람이 읽는 이름
    description: str
    patterns: list[str]              # 사용하는 패턴 이름 목록 (03-patterns.md 참조)
    market_condition: str            # "trending" | "ranging" | "volatile" | "any"
    timeframe: str                   # "intraday" | "swing" | "any"
    weight: float                    # 0.0 ~ 1.0 (우선순위 가중치)
    enabled: bool
    performance_score: float         # 자동 산출 (최근 성과)
    user_weight_override: float | None  # 사용자 직접 설정값 (None이면 auto)
    updated_at: datetime
```

### 기본 전략 목록

| ID | 이름 | 패턴 | 시장 조건 | 기본 가중치 |
|----|------|------|----------|------------|
| trend_follow | 추세추종 | golden_cross, ma_alignment, bull_flag | trending | 0.8 |
| reversal_major | 주요 반전 | head_and_shoulders, double_top, double_bottom | any | 0.75 |
| momentum_break | 모멘텀 돌파 | bollinger_breakout, macd_crossover | volatile | 0.7 |
| rsi_divergence | RSI 다이버전스 | rsi_divergence | ranging | 0.65 |
| pattern_pure | 순수 차트 패턴 | ascending_triangle, pennant, cup_handle | trending | 0.7 |
| volume_surge | 거래량 급등 | obv_trend + volume_ratio | any | 0.6 |

---

## 2. 우선순위 계산 알고리즘

### 최종 우선순위 점수

```python
def calculate_priority_score(signal: TradeSignal, strategy: Strategy) -> float:
    # 1. 신호 자체 점수 (패턴 confidence 가중 평균)
    signal_score = signal.score  # 0.0 ~ 1.0

    # 2. 전략 가중치 선택
    if strategy.user_weight_override is not None:
        strategy_weight = strategy.user_weight_override
    else:
        # 자동: 기본 가중치와 성과 점수 혼합
        strategy_weight = (
            strategy.weight * (1 - performance_blend_ratio)
            + strategy.performance_score * performance_blend_ratio
        )
        # performance_blend_ratio: 기본 0.3 (설정 가능)

    # 3. 시장 조건 보정
    market_condition_match = check_market_condition(strategy.market_condition)
    condition_multiplier = 1.2 if market_condition_match else 0.8

    # 4. 최종 점수
    return signal_score * strategy_weight * condition_multiplier
```

### 시장 조건 감지

```python
def check_market_condition(required: str) -> bool:
    """현재 시장이 전략이 요구하는 조건에 맞는지 확인"""
    if required == "any":
        return True

    adx = calculate_adx(market_index_ohlcv)  # 시장 지수 ADX

    current_condition = (
        "trending" if adx > 25
        else "volatile" if calculate_atr_ratio() > 1.5
        else "ranging"
    )
    return current_condition == required
```

---

## 3. 사용자 개입 방법

### 3.1 Admin UI를 통한 실시간 조정

Admin UI의 "전략 패널"에서:

```
[전략 목록]
┌─────────────────────────────────────────────────────────┐
│ 추세추종 (trend_follow)                    [활성화 토글] │
│ 현재 가중치: 0.80  성과 점수: 0.72  최종: AUTO          │
│ ──────────────────────────────────────────────────────  │
│ 사용자 오버라이드: [____0.90____]  [적용]  [자동으로]   │
│ 최근 7일 승률: 68%  평균 수익률: +2.3%                 │
└─────────────────────────────────────────────────────────┘
```

UI에서 변경 → Firebase Functions API 호출 → Firestore 업데이트
→ 다음 에이전트 실행 시 즉시 반영

### 3.2 Firebase Functions REST API

```
# 전략 가중치 조회
GET /api/strategies
GET /api/strategies/{strategyId}

# 전략 가중치 수정
PATCH /api/strategies/{strategyId}
Body: {
  "user_weight_override": 0.9,   // null이면 자동으로 복귀
  "enabled": true
}

# 종목 블랙리스트 추가
POST /api/blacklist
Body: { "ticker": "TSLA", "reason": "실적 발표 대기" }

# 전략 전체 리셋 (모든 user_weight_override를 null로)
POST /api/strategies/reset-overrides
```

인증: Firebase Auth Bearer Token 필요

### 3.3 설정 파일 정적 오버라이드

`config.yaml`에서 초기값 설정:

```yaml
strategies:
  overrides:
    trend_follow:
      enabled: true
      user_weight_override: 0.9
    reversal_major:
      enabled: false          # 비활성화
    momentum_break:
      user_weight_override: null  # 자동

blacklist:
  tickers:
    - "AMC"
    - "GME"
  sectors:
    - "Crypto ETF"
```

### 3.4 긴급 전체 중단

```
# Admin UI의 "비상 정지" 버튼 또는:
POST /api/control/pause
Body: { "reason": "시장 급락 대응" }

# 재개
POST /api/control/resume
```

비상 정지 시: Firestore `/config/system`의 `paused: true` 설정
에이전트 실행 시작 시 이 값을 먼저 확인.

---

## 4. 성과 기반 자동 가중치 조정

### 성과 지표 계산 (weekly-calibration 실행 시)

```python
def calculate_performance_score(strategy_id: str, window_days: int = 28) -> float:
    """최근 window_days일간의 전략 성과를 0~1 점수로 환산"""

    trades = load_trades(strategy_id, days=window_days)

    if len(trades) < min_trades_for_calibration:  # 기본 5회
        return strategy.weight  # 충분한 데이터 없으면 현재 가중치 유지

    win_rate = sum(1 for t in trades if t.pnl > 0) / len(trades)
    avg_pnl = mean(t.pnl_pct for t in trades)
    sharpe = calculate_sharpe(trades)

    # 정규화 (각 지표를 0~1 범위로)
    score = (
        win_rate * 0.4
        + normalize(avg_pnl, min=-0.05, max=0.10) * 0.4
        + normalize(sharpe, min=-1.0, max=3.0) * 0.2
    )
    return clip(score, 0.1, 1.0)
```

### 가중치 업데이트 규칙

```
새 가중치 = 이전 가중치 × 0.7 + 성과 점수 × 0.3

단, user_weight_override가 설정된 경우 → 자동 업데이트 스킵
(사용자 의도를 존중)

가중치 변동 제한:
  - 1회 업데이트에 ±0.2 이상 변동 금지 (급격한 변화 방지)
  - 최소값: 0.1 (완전히 0이 되지 않도록)
  - 최대값: 1.0
```

### 가중치 변경 이력

Firestore `/strategy_history` 컬렉션:
```
{historyId}:
  strategy_id: "trend_follow"
  changed_at: timestamp
  previous_weight: 0.80
  new_weight: 0.85
  change_source: "auto_calibration" | "user_override" | "config_file"
  performance_score: 0.78
  trades_evaluated: 12
```

---

## 5. 전략 우선순위 흐름도

```
PriorityAgent 실행
        │
        ▼
Firestore에서 전략 목록 로드
        │
        ▼
시스템 일시정지 여부 확인
  paused=true → 전체 종료
        │
        ▼
각 신호에 대해:
  ┌─────────────────────────────────────┐
  │ 해당 종목이 블랙리스트에 있는가?    │
  │   yes → 신호 제거                   │
  │   no  → 계속                        │
  └─────────────────────────────────────┘
        │
        ▼
  ┌─────────────────────────────────────┐
  │ 연관 전략이 enabled=false인가?      │
  │   yes → 신호 제거                   │
  │   no  → 계속                        │
  └─────────────────────────────────────┘
        │
        ▼
우선순위 점수 계산 (위 알고리즘)
        │
        ▼
점수 내림차순 정렬
        │
        ▼
상위 max_signals_per_run개 선택
        │
        ▼
filtered_signals → ExecutionAgent
```
