# 03 — 차트 패턴 인식 함수

## 개요

각 패턴은 독립적인 함수로 구현된다.
모든 함수는 동일한 시그니처를 따른다:

```python
def detect_<pattern_name>(
    ohlcv: pd.DataFrame,      # columns: open, high, low, close, volume
    config: PatternConfig,    # 패턴별 파라미터
) -> PatternResult | None:
    ...

@dataclass
class PatternResult:
    name: str               # 패턴명
    confidence: float       # 0.0 ~ 1.0
    signal: Literal["buy", "sell", "neutral"]
    detected_at_index: int  # OHLCV 데이터 내 감지 위치
    key_levels: dict        # 주요 가격 레벨 (목표가, 손절 참고)
    description: str        # 사람이 읽을 수 있는 설명
```

---

## 카테고리 1: 반전 패턴 (Reversal Patterns)

### 1.1 헤드앤숄더 (Head and Shoulders)

```
신호: sell (상승 후 반전)
함수: detect_head_and_shoulders(ohlcv, config)

알고리즘:
  1. 극값(피봇 포인트) 찾기 — scipy.signal.find_peaks 또는 직접 구현
  2. 3개 연속 고점 감지: 왼쪽어깨 < 머리 > 오른쪽어깨
  3. 어깨 높이 대칭성 확인: |left - right| / head < shoulder_asymmetry_threshold (기본 0.05)
  4. 넥라인(Neckline) 계산: 두 저점을 잇는 선
  5. 현재가가 넥라인 이탈 확인 → confidence 상향
  6. 거래량 확인: 머리 고점 시 거래량 < 왼쪽어깨 고점 시 거래량이면 confidence 상향

파라미터 (PatternConfig):
  - window: 패턴 탐색 기간 (기본 60봉)
  - min_head_height_pct: 어깨 대비 머리 최소 높이 비율 (기본 0.03)
  - shoulder_asymmetry_threshold: 0.05

신뢰도 산정:
  - 기본: 0.5
  - 넥라인 이탈: +0.2
  - 거래량 감소 확인: +0.15
  - 어깨 대칭성 양호: +0.15
```

### 1.2 역헤드앤숄더 (Inverse Head and Shoulders)

```
신호: buy (하락 후 반전)
함수: detect_inverse_head_and_shoulders(ohlcv, config)

알고리즘: detect_head_and_shoulders와 동일하나 저점 기준으로 탐색
```

### 1.3 이중천장 (Double Top)

```
신호: sell
함수: detect_double_top(ohlcv, config)

알고리즘:
  1. 최근 window 내 두 고점 탐색
  2. 두 고점 높이 차 < peak_tolerance_pct (기본 2%)
  3. 두 고점 사이 저점이 고점 대비 충분히 낮음 (기본 5% 이상)
  4. 두 번째 고점에서 거래량 < 첫 번째 고점 거래량 → confidence 상향
  5. 지지선(두 고점 사이 저점) 이탈 확인

파라미터:
  - window: 120봉
  - peak_tolerance_pct: 0.02
  - valley_depth_pct: 0.05
```

### 1.4 이중바닥 (Double Bottom)

```
신호: buy
함수: detect_double_bottom(ohlcv, config)

알고리즘: detect_double_top과 동일하나 저점 기준
```

### 1.5 삼중천장 / 삼중바닥 (Triple Top / Bottom)

```
신호: sell / buy
함수: detect_triple_top(ohlcv, config)
      detect_triple_bottom(ohlcv, config)

알고리즘: 이중천장/바닥 확장판. 3개 피크 탐색.
신뢰도: 이중 패턴 대비 +0.1 (더 강한 신호)
```

---

## 카테고리 2: 지속 패턴 (Continuation Patterns)

### 2.1 상승삼각형 (Ascending Triangle)

```
신호: buy (상승 지속)
함수: detect_ascending_triangle(ohlcv, config)

알고리즘:
  1. 저점이 꾸준히 상승하는지 확인 (선형 회귀 기울기 > 0)
  2. 고점이 수평 저항선을 형성하는지 확인 (기울기 ≈ 0)
  3. 저항선 돌파 확인 → confidence 대폭 상향
  4. 거래량이 수렴하다가 돌파 시 급증 → confidence 상향

신뢰도:
  - 패턴 형성만: 0.5
  - 저항선 돌파: +0.25
  - 거래량 확인: +0.15
```

### 2.2 하강삼각형 (Descending Triangle)

```
신호: sell (하락 지속)
함수: detect_descending_triangle(ohlcv, config)
알고리즘: 상승삼각형 반전
```

### 2.3 대칭삼각형 (Symmetrical Triangle)

```
신호: neutral → 돌파 방향에 따라 buy/sell
함수: detect_symmetrical_triangle(ohlcv, config)

알고리즘:
  1. 저점 상승 + 고점 하락 → 수렴 확인
  2. 돌파 방향 감지 후 신호 결정
```

### 2.4 깃발형 (Flag)

```
신호: 깃대 방향 지속 (상승 깃발 → buy, 하락 깃발 → sell)
함수: detect_bull_flag(ohlcv, config)
      detect_bear_flag(ohlcv, config)

알고리즘:
  1. 강한 추세 구간(깃대) 탐색: 단기간 큰 폭 상승/하락
  2. 이후 횡보/소폭 반대방향 구간(깃발) 확인
  3. 거래량: 깃대 구간 > 깃발 구간
  4. 깃발 이탈 확인

파라미터:
  - pole_min_change_pct: 깃대 최소 변동폭 (기본 5%)
  - flag_max_retracement: 깃발 최대 되돌림 (기본 50%)
```

### 2.5 페넌트 (Pennant)

```
신호: 깃대 방향 지속
함수: detect_pennant(ohlcv, config)
알고리즘: 깃발형과 유사, 깃발 부분이 대칭삼각형
```

### 2.6 쐐기형 (Wedge)

```
신호:
  - 상승 쐐기 (Rising Wedge) → sell
  - 하강 쐐기 (Falling Wedge) → buy
함수: detect_rising_wedge(ohlcv, config)
      detect_falling_wedge(ohlcv, config)

알고리즘:
  1. 고점/저점 모두 같은 방향으로 수렴
  2. 상승 쐐기: 고점/저점 모두 상승하나 저점 상승폭이 더 큼
  3. 쐐기 이탈 방향으로 신호 생성
```

---

## 카테고리 3: 이동평균 패턴

### 3.1 골든크로스 (Golden Cross)

```
신호: buy
함수: detect_golden_cross(ohlcv, config)

알고리즘:
  1. 단기 MA (기본 MA50)가 장기 MA (기본 MA200) 상향 돌파
  2. 교차 후 n봉 이내 (기본 5) 확인
  3. 교차 전 MA 간격이 좁아지다가 교차 → confidence 상향

파라미터:
  - short_window: 50
  - long_window: 200
  - cross_lookback: 5
```

### 3.2 데드크로스 (Death Cross)

```
신호: sell
함수: detect_death_cross(ohlcv, config)
알고리즘: 골든크로스 반전
```

### 3.3 이동평균 정배열 / 역배열

```
신호: buy (정배열) / sell (역배열)
함수: detect_ma_alignment(ohlcv, config)

알고리즘:
  정배열: MA5 > MA20 > MA60 > MA120
  역배열: MA5 < MA20 < MA60 < MA120
  정렬 강도: 각 MA 간격이 균등할수록 confidence 상향
```

---

## 카테고리 4: 모멘텀 패턴

### 4.1 RSI 다이버전스

```
신호:
  - 양의 다이버전스: 가격 저점 하락 + RSI 저점 상승 → buy
  - 음의 다이버전스: 가격 고점 상승 + RSI 고점 하락 → sell
함수: detect_rsi_divergence(ohlcv, config)

알고리즘:
  1. RSI 계산 (기본 14기간)
  2. 가격 피봇과 RSI 피봇을 비교
  3. 다이버전스 확인 (방향 불일치)

파라미터:
  - rsi_period: 14
  - pivot_lookback: 5
  - oversold_threshold: 30
  - overbought_threshold: 70
```

### 4.2 MACD 크로스오버

```
신호: buy (MACD가 Signal 상향돌파) / sell (하향돌파)
함수: detect_macd_crossover(ohlcv, config)

알고리즘:
  1. MACD = EMA12 - EMA26
  2. Signal = EMA9(MACD)
  3. 크로스오버 감지
  4. 히스토그램이 0선도 동시에 돌파하면 confidence 상향

파라미터:
  - fast_period: 12
  - slow_period: 26
  - signal_period: 9
```

### 4.3 볼린저 밴드 돌파 (Bollinger Breakout)

```
신호: buy (상단 돌파) / sell (하단 돌파)
함수: detect_bollinger_breakout(ohlcv, config)

알고리즘:
  1. 밴드 계산: 중간선 ± 2σ
  2. 스퀴즈 감지: 밴드 폭 < 최근 N일 최소폭
  3. 스퀴즈 후 돌파 시 confidence 대폭 상향

파라미터:
  - bb_period: 20
  - bb_std: 2.0
  - squeeze_lookback: 30
```

### 4.4 OBV 추세

```
신호: buy (OBV 상승 추세) / sell (OBV 하락 추세)
함수: detect_obv_trend(ohlcv, config)

알고리즘:
  1. OBV 계산
  2. OBV 단기/장기 이동평균 비교
  3. 가격은 횡보인데 OBV 상승 → 선행 매수 신호
```

---

## 카테고리 5: 캔들 패턴 (보조)

| 패턴명 | 함수 | 신호 |
|--------|------|------|
| 망치형 (Hammer) | detect_hammer | buy |
| 역망치형 (Inverted Hammer) | detect_inverted_hammer | buy |
| 도지 (Doji) | detect_doji | neutral (전환 주의) |
| 장악형 (Engulfing) | detect_engulfing | buy/sell |
| 별형 (Evening/Morning Star) | detect_star | buy/sell |
| 까마귀형 (Dark Cloud Cover) | detect_dark_cloud | sell |

캔들 패턴은 단독 신호보다 다른 패턴 신호의 보조 확인용으로 사용.
캔들 패턴 단독 confidence: 최대 0.5

---

## 패턴 함수 등록 및 관리

```python
PATTERN_REGISTRY = {
    # 반전 패턴
    "head_and_shoulders":         (detect_head_and_shoulders, PatternConfig(...)),
    "inverse_head_and_shoulders": (detect_inverse_head_and_shoulders, PatternConfig(...)),
    "double_top":                 (detect_double_top, PatternConfig(...)),
    "double_bottom":              (detect_double_bottom, PatternConfig(...)),
    "triple_top":                 (detect_triple_top, PatternConfig(...)),
    "triple_bottom":              (detect_triple_bottom, PatternConfig(...)),

    # 지속 패턴
    "ascending_triangle":         (detect_ascending_triangle, PatternConfig(...)),
    "descending_triangle":        (detect_descending_triangle, PatternConfig(...)),
    "symmetrical_triangle":       (detect_symmetrical_triangle, PatternConfig(...)),
    "bull_flag":                  (detect_bull_flag, PatternConfig(...)),
    "bear_flag":                  (detect_bear_flag, PatternConfig(...)),
    "pennant":                    (detect_pennant, PatternConfig(...)),
    "rising_wedge":               (detect_rising_wedge, PatternConfig(...)),
    "falling_wedge":              (detect_falling_wedge, PatternConfig(...)),

    # 이동평균
    "golden_cross":               (detect_golden_cross, PatternConfig(...)),
    "death_cross":                (detect_death_cross, PatternConfig(...)),
    "ma_alignment":               (detect_ma_alignment, PatternConfig(...)),

    # 모멘텀
    "rsi_divergence":             (detect_rsi_divergence, PatternConfig(...)),
    "macd_crossover":             (detect_macd_crossover, PatternConfig(...)),
    "bollinger_breakout":         (detect_bollinger_breakout, PatternConfig(...)),
    "obv_trend":                  (detect_obv_trend, PatternConfig(...)),
}

# 설정 파일에서 enabled_patterns 리스트로 활성화할 패턴 선택 가능
```

---

## 패턴 파라미터 오버라이드

각 패턴의 기본 파라미터는 `config.yaml`의 `patterns` 섹션에서 오버라이드 가능.
Firestore Admin API를 통해 런타임 중 변경 가능 (재시작 불필요).
