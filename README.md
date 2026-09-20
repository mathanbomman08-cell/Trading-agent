# QuantDock — Floating Trading Analysis Dock (Kotlin + Jetpack Compose)

Monica extension maadhiri, screen mela float aagura **control dock**. Endha app-la
irundhaalum bubble-a tap panna, prediction panel open aagum — price, signal,
confidence, target/stop, matrum "why this call" factor breakdown.

---

## 1. Open panna

1. Android Studio (Ladybug or newer) → **Open** → `QuantDock` folder.
2. Gradle sync (AGP 8.7.2, Kotlin 2.0.21, compileSdk 35, minSdk 26).
3. Run on a device/emulator.
4. App-la **Grant permission** press pannunga (Display over other apps) → **Start dock**.

Dock-a drag panni enga venaalum vaikalaam; position save aagum.

### Sync error vandha ("Task 'prepareKotlinBuildScriptModel' not found")

Idhu Gradle **version mismatch** — Studio local Gradle 9.x use panna AGP 8.7.2 work aagaadhu.
Ippo wrapper project-oda ship aagudhu (Gradle **8.11.1**). Idhu check pannunga:

1. **Settings → Build, Execution, Deployment → Build Tools → Gradle**
   - *Use Gradle from:* `gradle-wrapper.properties file`  ← idhu mukkiyam
   - *Gradle JDK:* **17** (JDK 21 um okay, 24 vendaam)
2. **File → Sync Project with Gradle Files**
3. Innum problem-na terminal-la: `./gradlew --stop` then `./gradlew clean assembleDebug`
4. Worst case: project root-la `.gradle/` and `.idea/` folder delete panni, marubadiyum open pannunga.

Version matrix ippo fixed-a iruku: Gradle 8.11.1 · AGP 8.7.2 · Kotlin 2.0.21 · JDK 17 · compileSdk 35.

## 2. Backend connect panradhu

`app/build.gradle.kts`:

```kotlin
buildConfigField("String", "API_BASE_URL", "\"http://10.0.2.2:8000/\"")
```

Expected endpoint (unga FastAPI platform-la add pannunga):

```
GET /api/v1/candles?symbol=XAUUSD&interval=1d&limit=250
```

```json
{
  "symbol": "XAUUSD",
  "interval": "1d",
  "candles": [
    {"t": 1717977600, "o": 2310.5, "h": 2325.0, "l": 2305.2, "c": 2321.4, "v": 148230}
  ]
}
```

FastAPI side sample:

```python
@app.get("/api/v1/candles")
def candles(symbol: str, interval: str = "1d", limit: int = 250):
    df = load_ohlcv(symbol, interval).tail(limit)
    return {
        "symbol": symbol,
        "interval": interval,
        "candles": [
            {"t": int(r.Index.timestamp()), "o": r.open, "h": r.high,
             "l": r.low, "c": r.close, "v": r.volume}
            for r in df.itertuples()
        ],
    }
}
```

Backend reach aagalana dock **offline simulated series**-la thodarum
(panel-la "offline data" badge varum), so UI test panna backend thevai illa.

## 3. Prediction engine

`domain/PredictionEngine.kt` — weighted ensemble, ovvoru factor-um `-1..+1`:

| Factor | Weight | Enna paakkudhu |
|---|---|---|
| Trend (EMA 20/50) | 0.25 | Trend structure, EMA separation % |
| Momentum (RSI 14) | 0.18 | Overbought/oversold + drive |
| MACD (12/26/9) | 0.17 | Histogram value + expanding/fading |
| Bollinger position | 0.12 | Mean-reversion stretch |
| Regression slope (30) | 0.18 | Least-squares slope, r² gated |
| Volume confirmation | 0.10 | Last bar vs 20-bar average |

- **Score** = weighted average → BULLISH / BEARISH / NEUTRAL (threshold ±0.12)
- **Confidence** = strength × factor agreement × volatility penalty (max 0.95)
- **Target / Stop** = ATR(14)-based (1.8× / 1.1×)
- **Forecast** = regression projection blended with the ensemble score

Ellaa indicator-um `domain/Indicators.kt`-la pure Kotlin-la iruku — no external TA library,
so unga backtesting platform-oda logic-a match panna easy-a edit pannalaam.

## 4. File map

```
app/src/main/java/com/quantdock/trading/
├── MainActivity.kt              permission + watchlist settings + start/stop
├── data/
│   ├── model/Models.kt          Candle, Prediction, Factor, DockState
│   ├── remote/MarketApi.kt      Retrofit interface
│   └── MarketRepository.kt      live fetch + offline fallback
├── domain/
│   ├── Indicators.kt            SMA, EMA, RSI, MACD, Bollinger, ATR, regression
│   └── PredictionEngine.kt      ensemble scoring
├── overlay/
│   ├── DockOverlayService.kt    foreground service + WindowManager
│   ├── OverlayViewOwner.kt      lifecycle owner for Compose in an overlay
│   ├── DockController.kt        state + auto-refresh loop
│   └── DockUi.kt                bubble + panel + sparkline
├── prefs/DockPrefs.kt
└── ui/theme/Theme.kt
```

## 5. Adutha step ideas

- Backtest tab: unga "Time Machine" endpoint-a dock-la irundhu trigger panna
- Alerts: score threshold thaandina notification
- Multi-symbol scanner view (dock-la ellaa symbol-um oru list-a)
- On-device ONNX model-a `PredictionEngine`-oda ensemble-la seventh factor-a serkkalaam

---

**Disclaimer:** Signals are model output for research, not investment advice.
Live money poduradhukku munnadi unga own backtest + risk rules use pannunga.
