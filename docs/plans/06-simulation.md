# 06 — 페이퍼 트레이딩 시뮬레이션

## 개요

실제 자금 없이 전략을 검증하는 가상 거래 시스템.
`mode: paper`로 실행 시 PaperBrokerAdapter를 통해 Firestore에 가상 거래를 기록한다.
실제 브로커 API와 동일한 인터페이스를 사용하므로 전환이 간단하다.

---

## 1. 브로커 어댑터 구조

```python
from abc import ABC, abstractmethod

class BrokerAdapter(ABC):
    """모든 브로커 어댑터가 구현해야 하는 인터페이스"""

    @abstractmethod
    def get_quote(self, ticker: str) -> Quote:
        """현재 시세 조회"""

    @abstractmethod
    def get_ohlcv(self, ticker: str, period: str, limit: int) -> pd.DataFrame:
        """OHLCV 시계열 조회"""

    @abstractmethod
    def place_order(self, order: OrderRequest) -> Order:
        """주문 제출"""

    @abstractmethod
    def get_position(self, ticker: str) -> Position | None:
        """현재 포지션 조회"""

    @abstractmethod
    def get_account(self) -> Account:
        """계좌 정보 조회"""

    @abstractmethod
    def cancel_order(self, order_id: str) -> bool:
        """주문 취소"""


class PaperBrokerAdapter(BrokerAdapter):
    """Firestore 기반 가상 거래 어댑터"""
    ...

class KisBrokerAdapter(BrokerAdapter):
    """한국투자증권 API 어댑터"""
    ...

class AlpacaBrokerAdapter(BrokerAdapter):
    """Alpaca API 어댑터"""
    ...
```

---

## 2. 페이퍼 계좌 모델

### Firestore 구조

```
/paper_account/{accountId}
  cash: 10000000.0              # 초기 자금 (설정값)
  total_value: 10000000.0       # 현재 총 평가액
  positions_value: 0.0          # 보유 포지션 평가액
  realized_pnl: 0.0             # 실현 손익
  unrealized_pnl: 0.0           # 미실현 손익
  created_at: timestamp
  updated_at: timestamp

/paper_positions/{positionId}
  account_id: "default"
  ticker: "AAPL"
  qty: 10
  avg_entry_price: 175.50
  current_price: 180.00
  unrealized_pnl: 45.00
  unrealized_pnl_pct: 0.0256
  opened_at: timestamp
  strategy_id: "trend_follow"
  stop_loss: 170.00
  target: 186.00

/paper_orders/{orderId}
  account_id: "default"
  ticker: "AAPL"
  side: "buy"           # "buy" | "sell"
  order_type: "market"  # "market" | "limit"
  qty: 10
  limit_price: null     # 시장가 주문 시 null
  requested_price: 175.50
  executed_price: 175.85  # 슬리피지 적용
  status: "filled"
  strategy_id: "trend_follow"
  signal_id: "sig_20240315_001"
  created_at: timestamp
  executed_at: timestamp
  commission: 0.0       # 수수료 (설정 가능)
```

---

## 3. 슬리피지 모델

실제 거래와 유사한 환경을 만들기 위해 슬리피지를 시뮬레이션한다.

```python
def simulate_slippage(
    requested_price: float,
    side: Literal["buy", "sell"],
    qty: int,
    volume: float,
    slippage_config: SlippageConfig,
) -> float:
    """
    슬리피지 시뮬레이션

    모델: 거래량 비례 슬리피지
    - 기본 슬리피지: base_slippage_pct (기본 0.05%)
    - 거래량 충격: qty / daily_volume × impact_factor
    """
    base = slippage_config.base_slippage_pct

    # 거래량 충격
    volume_impact = (qty / volume) * slippage_config.impact_factor

    total_slippage = base + volume_impact
    total_slippage = min(total_slippage, slippage_config.max_slippage_pct)

    if side == "buy":
        return requested_price * (1 + total_slippage)
    else:
        return requested_price * (1 - total_slippage)
```

### SlippageConfig 기본값

```yaml
paper_trading:
  slippage:
    base_slippage_pct: 0.0005   # 0.05%
    impact_factor: 2.0
    max_slippage_pct: 0.01      # 최대 1%
  commission:
    per_trade: 0.0              # 무료 (설정 가능)
    per_share: 0.0
    min_commission: 0.0
```

---

## 4. PaperBrokerAdapter 구현 계획

```python
class PaperBrokerAdapter(BrokerAdapter):

    def __init__(self, db: FirestoreClient, account_id: str, quote_source: BrokerAdapter):
        self.db = db
        self.account_id = account_id
        self.quote_source = quote_source  # 실제 시세는 외부 소스에서

    def get_quote(self, ticker: str) -> Quote:
        # 실제 시세 소스에서 가져옴 (공개 API 또는 브로커 API 읽기 전용)
        return self.quote_source.get_quote(ticker)

    def get_ohlcv(self, ticker: str, period: str, limit: int) -> pd.DataFrame:
        return self.quote_source.get_ohlcv(ticker, period, limit)

    def place_order(self, order: OrderRequest) -> Order:
        # 1. 계좌 잔고 확인
        account = self.get_account()
        required_cash = order.qty * order.requested_price

        if order.side == "buy" and account.cash < required_cash:
            raise InsufficientFundsError(f"잔고 부족: {account.cash:.0f} < {required_cash:.0f}")

        # 2. 현재 시세 확인
        quote = self.get_quote(order.ticker)

        # 3. 슬리피지 계산
        executed_price = simulate_slippage(
            requested_price=quote.price,
            side=order.side,
            qty=order.qty,
            volume=quote.volume,
            slippage_config=self.slippage_config,
        )

        # 4. Firestore 업데이트 (트랜잭션)
        with self.db.transaction() as txn:
            self._update_account(txn, order, executed_price)
            self._update_position(txn, order, executed_price)
            return self._create_order_record(txn, order, executed_price)

    def get_account(self) -> Account:
        doc = self.db.collection("paper_account").document(self.account_id).get()
        return Account(**doc.to_dict())
```

---

## 5. 시뮬레이션 vs 라이브 전환

### 설정 방법

```yaml
# config.yaml
trading:
  mode: "paper"  # "paper" | "live"
  paper:
    account_id: "default"
    initial_cash: 10000000
    reset_on_start: false   # true면 매 실행마다 계좌 초기화
  live:
    broker: "samsung"       # "samsung" (삼성증권)
    confirmed: false        # 라이브 전환 확인 플래그 (반드시 수동 설정)
```

### 시뮬레이션 모드 핵심 원칙

```
시뮬레이션 모드(paper)에서는 절대로 실제 매매가 발생하지 않는다.

보장 방법:
  1. mode=paper → PaperBrokerAdapter만 생성 (브로커 API 연결 자체 안 함)
  2. PaperBrokerAdapter.place_order()는 Firestore에만 기록
  3. 삼성증권 API의 주문 엔드포인트는 호출되지 않음
  4. 시세 조회는 공개 API 우선 → 삼성증권 API 폴백 (읽기 전용)
```

### 데이터 소스와 모드의 관계

```
┌──────────────────┬──────────────────┬──────────────────┐
│ 기능              │ paper 모드       │ live 모드        │
├──────────────────┼──────────────────┼──────────────────┤
│ 시세 조회 (일봉)  │ 공개 API         │ 공개 API         │
│ 시세 조회 (분봉)  │ 공개 API → 삼성증권│ 공개 API → 삼성증권│
│ 종목 정보         │ 공개 API         │ 공개 API         │
│ 매매 주문         │ Firestore 기록만 │ 삼성증권 API      │
│ 계좌 잔고         │ Firestore 가상계좌│ 삼성증권 API      │
│ 포지션 조회       │ Firestore        │ 삼성증권 API      │
└──────────────────┴──────────────────┴──────────────────┘
```

### 전환 보호 장치 (다중 안전장치)

```python
def get_broker_adapter(config: Config) -> BrokerAdapter:
    # 공개 API 기반 시세 소스 (모든 모드에서 1순위)
    public_quote = PublicQuoteProvider(config)

    if config.trading.mode == "paper":
        # 페이퍼 모드: 브로커 API에 주문 기능 연결 안 함
        quote_source = FallbackQuoteProvider(
            primary=public_quote,
            fallback=SamsungQuoteProvider(config) if config.brokers.samsung.enabled else None
        )
        return PaperBrokerAdapter(db, config.trading.paper.account_id, quote_source)

    # 라이브 모드: 다중 확인 단계
    if config.trading.mode == "live":
        # 안전장치 1: config 명시적 확인 플래그
        if not config.trading.live.confirmed:
            raise LiveModeNotConfirmedError(
                "라이브 모드 전환을 위해 config.yaml에서 "
                "trading.live.confirmed: true 를 설정하세요."
            )

        # 안전장치 2: 환경 변수 이중 확인
        if os.environ.get("ENABLE_LIVE_TRADING") != "true":
            raise LiveModeNotConfirmedError(
                "환경 변수 ENABLE_LIVE_TRADING=true 가 설정되지 않았습니다."
            )

        # 안전장치 3: Firestore 시스템 설정에서도 확인
        system_config = db.collection("config").document("system").get()
        if not system_config.to_dict().get("live_trading_enabled", False):
            raise LiveModeNotConfirmedError(
                "Firestore config/system의 live_trading_enabled가 false입니다. "
                "Admin UI에서 라이브 거래를 활성화하세요."
            )

        logger.warning("⚠️  라이브 거래 모드로 실행됩니다!")
        return SamsungBrokerAdapter(config, public_quote)
```

---

## 6. 백테스트 (확장 기능)

페이퍼 트레이딩과 별도로 과거 데이터 기반 백테스트 지원 계획.

```
BacktestRunner:
  - 입력: 과거 OHLCV 데이터, 전략 설정
  - 동일한 LangGraph 그래프를 과거 데이터에 대해 반복 실행
  - 각 시점에서 브로커 어댑터를 BacktestBrokerAdapter로 교체
  - 결과: 수익률 곡선, 최대 낙폭, 샤프지수, 승률
  - Admin UI에서 백테스트 결과 시각화
```

```yaml
# 백테스트 설정 예시
backtest:
  enabled: false
  start_date: "2023-01-01"
  end_date: "2023-12-31"
  initial_cash: 10000000
  strategies:
    - "trend_follow"
    - "reversal_major"
```

---

## 7. 성과 리포트

### 일일 리포트 (post_market_analysis 시 생성)

```
/reports/{date}
  date: "2024-03-15"
  mode: "paper"
  total_trades: 5
  winning_trades: 3
  win_rate: 0.60
  total_pnl: 125000
  total_pnl_pct: 1.25
  best_trade: {ticker: "AAPL", pnl: 89000}
  worst_trade: {ticker: "META", pnl: -32000}
  strategy_breakdown:
    trend_follow: {trades: 3, win_rate: 0.67, pnl: 150000}
    momentum_break: {trades: 2, win_rate: 0.50, pnl: -25000}
  portfolio_value_history: [...]
```

Admin UI 대시보드에서 리포트 조회 및 차트 시각화.
