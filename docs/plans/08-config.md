# 08 — 설정 파일 스키마

## 개요

모든 설정은 `config/config.yaml`에 정의한다.
민감한 값(API 키, 비밀번호)은 GCP Secret Manager에 저장하고
config.yaml에서 `secret://SECRET_NAME` 형식으로 참조한다.

config.yaml은 `.gitignore`에 포함. `config.example.yaml`을 템플릿으로 커밋.

---

## 전체 설정 파일 스키마

```yaml
# config/config.yaml
# ─────────────────────────────────────────────
# 시스템 전역 설정
# ─────────────────────────────────────────────
system:
  project_id: "my-stock-project"       # GCP 프로젝트 ID
  region: "asia-northeast3"            # GCP 리전
  log_level: "INFO"                    # DEBUG | INFO | WARNING | ERROR
  timezone: "Asia/Seoul"               # 실행 타임존
  paused: false                        # 비상 정지 플래그 (Admin UI에서도 제어)

# ─────────────────────────────────────────────
# 거래 설정
# ─────────────────────────────────────────────
trading:
  mode: "paper"                        # "paper" | "live"
  market: "KR"                         # "KR" | "US"

  paper:
    account_id: "default"
    initial_cash: 10000000             # 초기 가상 자금 (원)
    reset_on_start: false              # 실행마다 계좌 초기화 여부
    slippage:
      base_slippage_pct: 0.0005        # 기본 슬리피지 0.05%
      impact_factor: 2.0               # 거래량 충격 계수
      max_slippage_pct: 0.01           # 최대 슬리피지 1%
    commission:
      per_trade: 0.0                   # 거래당 수수료
      per_share: 0.0                   # 주당 수수료
      rate: 0.00015                    # 비율 수수료 (0.015%)

  live:
    confirmed: false                   # 라이브 전환 확인 플래그 (true 설정 필수)
    broker: "samsung"                  # "samsung" (삼성증권)

# ─────────────────────────────────────────────
# 데이터 소스 설정 (공개 API 우선)
# ─────────────────────────────────────────────
data_sources:
  # 공개 API 설정 (1순위, 무료)
  public_apis:
    yfinance:
      enabled: true                    # Yahoo Finance (글로벌 일봉, 기본 지표)
    krx:
      enabled: true                    # KRX 정보데이터시스템 (한국 종목 정보)
    naver_finance:
      enabled: true                    # 네이버 금융 (한국 종목 보조 데이터)
    fred:
      enabled: false                   # FRED (매크로 지표, 필요 시 활성화)
      api_key: "secret://fred-api-key" # 무료 API 키

  # 공개 API → 브로커 API 자동 폴백 활성화
  fallback_enabled: true               # true: 공개 API 실패 시 삼성증권 API로 폴백

# ─────────────────────────────────────────────
# 브로커 API 설정 (삼성증권, 매매 + 폴백 시세)
# ─────────────────────────────────────────────
brokers:
  samsung:
    enabled: true                      # 삼성증권 API 사용 여부
    app_key: "secret://samsung-app-key"
    app_secret: "secret://samsung-app-secret"
    account_no: "secret://samsung-account-no"
    base_url: "https://openapi.samsungpop.com"
    is_real: false                     # false = 모의투자

# ─────────────────────────────────────────────
# LLM 설정 (선택적)
# ─────────────────────────────────────────────
llm:
  enabled: false                       # LLM 사용 여부
  provider: "openai"                   # "openai" | "anthropic" | "custom"
  api_key: "secret://llm-api-key"
  api_base_url: null                   # custom 시 엔드포인트 URL
  model: "gpt-4o-mini"                 # 사용 모델명
  timeout: 30                          # 요청 타임아웃 (초)
  max_retries: 2

  # 어떤 에이전트에서 LLM을 활성화할지
  agent_usage:
    scanner: false                     # 뉴스 기반 이상 감지
    pattern: false                     # 차트 이미지 패턴 검증
    signal: false                      # 신호 최종 검토

  # custom provider 예시
  # provider: "custom"
  # api_base_url: "http://localhost:11434/v1"  # Ollama
  # model: "llama3.1"

# ─────────────────────────────────────────────
# 종목 우주 설정
# ─────────────────────────────────────────────
universe:
  watchlist:                           # 항상 스캔할 종목 (Level 1)
    - "005930"                         # 삼성전자
    - "000660"                         # SK하이닉스
    - "035420"                         # NAVER
    - "051910"                         # LG화학

  core_universe_source: "kospi200"     # "kospi200" | "sp500" | "custom"
  custom_core:                         # core_universe_source: custom 시
    - "..."

  extended_universe:
    enabled: true
    size: 300                          # 확장 우주 종목 수
    update_schedule: "weekly"          # 업데이트 주기

# ─────────────────────────────────────────────
# 스캔 필터 설정
# ─────────────────────────────────────────────
filters:
  min_volume_ratio: 1.2                # 20일 평균 대비 최소 거래량 비율
  min_price_change_pct: 0.005          # 최소 일중 변동폭 0.5%
  min_market_cap: 100000000000        # 최소 시가총액 (원, 1천억)
  rsi_min: 25                          # RSI 최소값 (이 미만은 과매도 제외)
  rsi_max: 75                          # RSI 최대값 (이 초과는 과매수 제외)
  ma_filter_enabled: true              # 이동평균 필터 활성화

# ─────────────────────────────────────────────
# 패턴 인식 설정
# ─────────────────────────────────────────────
patterns:
  enabled_patterns:                    # 활성화할 패턴 목록 (03-patterns.md 참조)
    - "golden_cross"
    - "dead_cross"
    - "ma_alignment"
    - "macd_crossover"
    - "rsi_divergence"
    - "bollinger_breakout"
    - "double_bottom"
    - "double_top"
    - "head_and_shoulders"
    - "bull_flag"
    - "ascending_triangle"

  min_confidence: 0.60                 # 최소 신뢰도 임계값

  # 패턴별 파라미터 오버라이드
  overrides:
    golden_cross:
      short_window: 50
      long_window: 200
    rsi_divergence:
      rsi_period: 14
      oversold_threshold: 30
      overbought_threshold: 70

# ─────────────────────────────────────────────
# 신호 생성 설정
# ─────────────────────────────────────────────
signals:
  buy_threshold: 0.70                  # 매수 신호 최소 점수
  sell_threshold: 0.70                 # 매도 신호 최소 점수
  max_signals_per_run: 5               # 실행당 최대 신호 수

# ─────────────────────────────────────────────
# 리스크 관리 설정
# ─────────────────────────────────────────────
risk:
  max_position_pct: 0.05               # 단일 포지션 최대 비율 5%
  max_portfolio_exposure: 0.50         # 총 포지션 최대 비율 50%
  max_sector_pct: 0.20                 # 동일 섹터 최대 비율 20%
  max_open_positions: 10               # 최대 동시 보유 종목 수
  kelly_fraction: 0.10                 # Kelly Criterion 분수
  atr_multiplier: 2.0                  # 손절가 ATR 배수
  rr_ratio: 2.0                        # Risk-Reward 비율
  max_holding_days: 20                 # 최대 보유 기간 (스윙)
  max_slippage: 0.005                  # 신호가 대비 최대 슬리피지 0.5%

# ─────────────────────────────────────────────
# 전략 우선순위 설정
# ─────────────────────────────────────────────
strategies:
  performance_blend_ratio: 0.30        # 성과 점수 반영 비율
  min_trades_for_calibration: 5        # 자동 조정 최소 거래 수
  weight_update_max_delta: 0.20        # 1회 최대 가중치 변동
  min_weight: 0.10
  max_weight: 1.00

  overrides:                           # 특정 전략 가중치 직접 지정
    trend_follow:
      enabled: true
      user_weight_override: null       # null이면 자동
    reversal_major:
      enabled: true
      user_weight_override: null

  blacklist:
    tickers: []                        # 거래 제외 종목
    sectors: []                        # 거래 제외 섹터

# ─────────────────────────────────────────────
# 알림 설정
# ─────────────────────────────────────────────
notifications:
  email:
    enabled: false
    provider: "sendgrid"
    api_key: "secret://sendgrid-api-key"
    from: "bot@example.com"
    to:
      - "admin@example.com"

  slack:
    enabled: false
    webhook_url: "secret://slack-webhook-url"

  alert_on:
    - "order_executed"
    - "stop_loss_hit"
    - "error"
    - "weekly_report"

# ─────────────────────────────────────────────
# Admin UI 설정
# ─────────────────────────────────────────────
admin_ui:
  allowed_emails:
    - "admin@example.com"
  log_retention_days: 30               # 로그 보관 기간
  max_run_history: 100                 # UI에 표시할 최대 실행 이력
```

---

## 설정 로딩 코드 (Python)

```python
import yaml
from google.cloud import secretmanager
from pydantic import BaseModel

def load_config(path: str = "config/config.yaml") -> Config:
    with open(path) as f:
        raw = yaml.safe_load(f)

    # Secret Manager 참조 해석
    resolved = resolve_secrets(raw)

    # Pydantic 검증
    return Config.model_validate(resolved)

def resolve_secrets(obj: Any) -> Any:
    """secret://SECRET_NAME 형식의 값을 Secret Manager에서 로드"""
    if isinstance(obj, str) and obj.startswith("secret://"):
        secret_name = obj[len("secret://"):]
        return _fetch_secret(secret_name)
    elif isinstance(obj, dict):
        return {k: resolve_secrets(v) for k, v in obj.items()}
    elif isinstance(obj, list):
        return [resolve_secrets(v) for v in obj]
    return obj

def _fetch_secret(secret_name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{PROJECT_ID}/secrets/{secret_name}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")
```

---

## 환경별 설정 오버라이드

```bash
# 개발 환경
CONFIG_PATH=config/config.dev.yaml python main.py

# 스테이징
CONFIG_PATH=config/config.staging.yaml python main.py

# 프로덕션 (기본)
python main.py
```

Cloud Run Job에서는 환경 변수로 전달:
```yaml
# Cloud Run Job 환경 변수
env:
  - name: CONFIG_PATH
    value: "config/config.yaml"
  - name: GCP_PROJECT_ID
    value: "my-stock-project"
```
