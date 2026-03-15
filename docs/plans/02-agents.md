# 02 — Agent 정의 및 결정 로직

## 개요

LangGraph의 각 노드가 하나의 Agent에 해당한다.
모든 Agent는 **알고리즘 기반 결정**을 기본으로 하며,
설정에서 `llm_enabled: true`일 때만 LLM을 보조적으로 사용한다.

## LangGraph 상태 (GraphState)

```python
class GraphState(TypedDict):
    # 실행 컨텍스트
    run_id: str
    mode: Literal["paper", "live"]
    timestamp: datetime

    # 스캔 대상
    universe: list[str]           # 전체 종목 티커 목록
    candidates: list[StockCandidate]

    # 패턴/신호
    pattern_results: dict[str, list[PatternResult]]
    signals: list[TradeSignal]

    # 리스크/우선순위 적용 후
    filtered_signals: list[TradeSignal]
    sized_signals: list[SizedSignal]   # 포지션 사이징 완료

    # 실행 결과
    executed_orders: list[Order]

    # 모니터링
    open_positions: list[Position]
    exit_signals: list[ExitSignal]

    # 메타
    node_logs: dict[str, NodeLog]
    errors: list[str]
```

---

## Agent 1: ScannerAgent

**역할**: 종목 우주(universe)에서 1차 후보 종목을 추출

### 결정 기준 (알고리즘)
1. Firestore `/universe`에서 enabled=true 종목 로드
2. 데이터 소스 계층에 따라 시세 데이터 조회:
   - 1순위: 공개 API (yfinance, KRX 정보데이터시스템)
   - 2순위: 삼성증권 API (공개 API 실패 또는 분봉 데이터 필요 시)
3. 기초 필터 통과 기준:
   - 거래량 > 20일 평균 거래량 × `min_volume_ratio` (기본 1.2)
   - 일중 변동폭 > `min_price_change_pct` (기본 0.5%)
   - 시가총액 > `min_market_cap` (설정값)
4. 통과 종목을 `candidates`로 상태에 저장

### LLM 보조 (옵션)
- 뉴스 헤드라인 요약 → 섹터별 이상 징후 감지
- 프롬프트: "다음 종목들 중 오늘 주목할 섹터 리스크가 있는 종목을 알려줘"
- LLM 결과로 특정 종목 제외 또는 가중치 하향

### 출력
```python
candidates: list[StockCandidate]  # ticker, price_data, volume_ratio, change_pct
```

---

## Agent 2: FilterAgent

**역할**: 후보 종목에서 기술적 조건으로 추가 필터링

### 결정 기준 (알고리즘)
1. 이동평균 정배열/역배열 확인 (MA5 > MA20 > MA60)
2. RSI 과매수/과매도 상태 확인 (기본: 30 < RSI < 70 통과)
3. 볼린저 밴드 상대 위치 계산
4. 최근 N일 지지/저항선 존재 여부
5. 설정된 필터 조건 모두 통과한 종목만 유지

### 필터 우선순위 적용
- 전략 가중치(StrategyPriorityAgent에서 로드)에 따라 적용할 필터 선택
- 예: 추세추종 전략 우선 → MA 정배열 필터 강화

### 출력
```python
candidates: list[StockCandidate]  # 필터 통과 종목만
```

---

## Agent 3: PatternAgent

**역할**: 후보 종목의 차트에서 패턴을 인식

### 결정 기준 (알고리즘)
각 패턴 인식 함수를 병렬 적용. 상세 함수 목록은 [03-patterns.md](./03-patterns.md) 참조.

```
적용 패턴 카테고리:
  - 반전 패턴: 헤드앤숄더, 이중바닥/이중천장, 삼중바닥/삼중천장
  - 지속 패턴: 삼각형, 깃발형, 페넌트, 쐐기형
  - 이동평균 패턴: 골든크로스, 데드크로스, 정배열
  - 모멘텀 패턴: RSI 다이버전스, MACD 크로스, 볼린저 돌파
  - 거래량 패턴: OBV 상승, 거래량 급증 후 안정
```

### 패턴 신뢰도 점수
각 패턴 함수는 `confidence: float (0~1)` 반환.
`confidence < min_confidence` (기본 0.6) 패턴은 제외.

### LLM 보조 (옵션)
- 차트 이미지(캔들스틱) 생성 → LLM Vision API로 패턴 재확인
- 설정: `llm_pattern_validation: true`
- LLM이 동의하지 않으면 confidence를 0.8배로 조정

### 출력
```python
pattern_results: dict[str, list[PatternResult]]
# {"AAPL": [PatternResult(name="golden_cross", confidence=0.85, signal="buy")], ...}
```

---

## Agent 4: SignalAgent

**역할**: 패턴 결과를 취합하여 매매 신호 생성

### 결정 기준 (알고리즘)
1. 종목별 패턴 결과를 집계
2. 매수 신호 점수 = Σ(패턴별 confidence × 패턴 가중치)
3. 매도 신호 점수 = Σ(반전 패턴 confidence × 패턴 가중치)
4. 신호 생성 조건:
   - 매수: buy_score > `signal_buy_threshold` (기본 0.7)
   - 매도: sell_score > `signal_sell_threshold` (기본 0.7)
5. 복수 패턴 일치 시 신호 강도(strength) 상향

### 신호 충돌 처리
- 동일 종목에 매수/매도 신호 동시 → 더 높은 점수 채택, 로그 기록
- 기존 보유 포지션 있으면 → 매수 신호 무시, 매도 신호 우선

### LLM 보조 (옵션)
- 패턴 결과 + 시장 컨텍스트 → LLM에게 신호 최종 검토 요청
- 프롬프트: "다음 패턴들을 고려할 때 매매 신호의 신뢰도를 평가해줘"

### 출력
```python
signals: list[TradeSignal]
# TradeSignal(ticker, side, strength, patterns[], score, reason)
```

---

## Agent 5: RiskAgent

**역할**: 리스크 평가 및 포지션 사이징

### 결정 기준 (알고리즘)
1. **포지션 사이징** (Kelly Criterion 변형):
   - `position_size = account_equity × kelly_fraction × signal_strength`
   - `kelly_fraction` 기본 0.1 (보수적)
2. **최대 단일 포지션** 제한: `max_position_pct` (기본 5%)
3. **최대 포트폴리오 비중** 제한: `max_portfolio_exposure` (기본 50%)
4. **손절가 계산**:
   - ATR 기반: `stop_loss = entry - (ATR × atr_multiplier)`
   - 기본 `atr_multiplier = 2.0`
5. **목표가 계산**:
   - Risk-Reward 비율 기반: `target = entry + (entry - stop) × rr_ratio`
   - 기본 `rr_ratio = 2.0`
6. 리스크 조건 미충족 시 해당 신호 제거

### 포트폴리오 레벨 제한
- 동일 섹터 편중 방지: 섹터당 최대 `max_sector_pct` (기본 20%)
- 개방 포지션 수 제한: `max_open_positions` (기본 10개)

### 출력
```python
sized_signals: list[SizedSignal]
# SizedSignal(ticker, side, qty, entry_price, stop_loss, target, risk_amount)
```

---

## Agent 6: PriorityAgent

**역할**: 전략 우선순위를 적용하여 최종 실행 목록 결정

### 결정 기준 (알고리즘)
1. Firestore `/strategies`에서 전략별 가중치 로드
2. 각 신호에 연관된 패턴/전략의 가중치 적용
3. 우선순위 점수 = `signal_score × strategy_weight`
4. 점수 내림차순 정렬
5. 상위 `max_signals_per_run`개 (기본 5개)만 실행 목록에 포함
6. 사용자 직접 설정 (`user_override`) 확인:
   - 특정 전략 비활성화 → 해당 전략 신호 전부 제거
   - 특정 종목 블랙리스트 → 해당 종목 신호 제거
   - 전략 강제 활성화 → 해당 전략 점수 최고값으로 설정

### 출력
```python
filtered_signals: list[SizedSignal]  # 우선순위 적용 후 최종 목록
```

---

## Agent 7: ExecutionAgent

**역할**: 주문 실행 또는 시뮬레이션

### 결정 기준 (알고리즘)
1. `mode` 확인: `paper` → PaperBrokerAdapter, `live` → 실제 브로커
2. 각 신호에 대해:
   - 현재 시세 재확인 (슬리피지 방지)
   - 진입가 괴리 확인: `|current_price - signal_price| / signal_price > max_slippage`이면 보류
   - 주문 생성 → 브로커 어댑터로 전송
3. 주문 결과 Firestore `/orders`에 기록
4. 성공한 주문은 `/positions`에 기록

### 페이퍼 트레이딩 처리
- `/paper_account`에서 현금 차감
- 슬리피지 시뮬레이션: `executed_price = signal_price × (1 + slippage_pct)`
- 상세 내용: [06-simulation.md](./06-simulation.md)

### 출력
```python
executed_orders: list[Order]
```

---

## Agent 8: MonitorAgent

**역할**: 기존 보유 포지션의 청산 조건 모니터링

### 결정 기준 (알고리즘)
1. Firestore `/positions`에서 open 포지션 로드
2. 각 포지션에 대해:
   - 현재가 < 손절가 → 즉시 매도 신호
   - 현재가 > 목표가 → 매도 신호 (또는 트레일링 스탑 전환)
   - 보유 기간 > `max_holding_days` → 강제 청산 신호
   - 패턴 반전 감지 → 조기 청산 신호
3. 트레일링 스탑 업데이트 (매일 최고가 갱신)

### 출력
```python
exit_signals: list[ExitSignal]  # ExecutionAgent로 전달되어 청산 주문 생성
```

---

## Agent 실행 그래프 구조

```
START
  │
  ▼
[ScannerAgent] ──candidates──► [FilterAgent]
                                     │
                               candidates (filtered)
                                     │
                                     ▼
                               [PatternAgent]
                                     │
                               pattern_results
                                     │
                                     ▼
                               [SignalAgent]
                                     │
                                signals
                                     │
                          ┌──────────┴──────────┐
                          ▼                     ▼
                    [RiskAgent]          [MonitorAgent]
                          │                     │
                    sized_signals          exit_signals
                          │                     │
                          └──────────┬──────────┘
                                     ▼
                              [PriorityAgent]
                                     │
                               filtered_signals
                                     │
                                     ▼
                              [ExecutionAgent]
                                     │
                               executed_orders
                                     │
                                    END
```

### 조건부 엣지
- `candidates` 비어 있음 → END (스캔 대상 없음)
- `signals` 비어 있음 → MonitorAgent만 실행 후 END
- `mode == live` && `is_market_closed()` → END

---

## Agent별 LLM 사용 요약

| Agent | LLM 사용 | 사용 목적 |
|-------|----------|----------|
| ScannerAgent | 선택적 | 뉴스 기반 이상 종목 감지 |
| PatternAgent | 선택적 | 차트 이미지 패턴 재검증 |
| SignalAgent | 선택적 | 신호 신뢰도 최종 검토 |
| RiskAgent | 미사용 | 수식 기반으로 충분 |
| PriorityAgent | 미사용 | 가중치 기반으로 충분 |
| ExecutionAgent | 미사용 | 결정적 실행 필요 |
| MonitorAgent | 미사용 | 수식 기반으로 충분 |
| FilterAgent | 미사용 | 조건 기반으로 충분 |
