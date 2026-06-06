---
name: project-overview
description: "OPTIX Android stock sentiment app — full stack, architecture, CI/CD, version, feature inventory, and known gaps as of May 2026"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8f0f0737-d25e-4bc0-9ca8-a19b3fab1e11
---

# OPTIX — Android Stock Sentiment App

## Identity
- **App ID**: `com.maverickideas.optix` | Package: `com.maverickideas.stocksentiment`
- **Version**: 1.0.2 (versionCode=3) as of May 2026
- **Min SDK**: 26 | Target/Compile SDK: 35
- **Language**: Kotlin | **UI**: Jetpack Compose + Material3

## Tech Stack
- **Auth**: Firebase Auth (Email/Password + Google Sign-In via Credential Manager)
- **Backend**: Firestore (real-time listeners), Firebase Cloud Functions (AI + API proxying), Firebase Analytics + Crashlytics
- **AI**: Claude Sonnet 4.5 via Cloud Functions — predictions, strategy recommendations
- **Data**: Polygon.io (stocks/options, server-side proxy), Yahoo Finance (crypto fallback), RSS news feeds
- **Local**: DataStore (search history), SharedPreferences (theme, tutorial, location decision)
- **CI/CD**: Bitrise → Git Flow → Google Play Internal Track
- Gradle 8.5.2, Kotlin 1.9.x, Compose BOM 2024.09.03, Firebase BOM 33.5.1

## Architecture
Clean layer separation: `domain/` → `data/` → `ui/`

- **domain/**: SignalEngine (rule-based), AiStrategyEngine (Claude), PredictionEngine (Claude + rule fallback), SentimentAnalyzer, CryptoSignalEngine, AiSignalEngine (legacy?)
- **data/**: Firebase repos (Auth, User, Watch, Feedback), data repos (Stock, Crypto, News, CryptoNews, Polygon), location repo, YahooFinanceService, DataStore (search history)
- **ui/screens**: Search (home), Dashboard (stock), CryptoDashboard, MacroStocks, MacroCrypto, Auth, Terms, Admin, WorldMapScreen (unfinished)
- **ui/viewmodels**: Auth, Stock, Crypto, Home, Watch, Location, Admin
- **ui/components**: ~30 reusable Compose components (charts, cards, sheets, marquees)
- **ui/tutorial**: Spotlight wizard system (TutorialController, TutorialOverlay, TutorialFlows, TutorialPrefs)
- **ui/theme**: Material3 dark/light + custom OPTIX color palette
- **util**: Telemetry, AlertNotifier, MarketStatus, SymbolDetector, Numbers, NonFatalExceptions

## Firestore Schema
- `users/{uid}` — profile (role: user|admin, city, region, country, hasAcceptedTos, tosAcceptedAt)
- `users/{uid}/watchlist/{symbol}` — WatchlistItem (max 50 per STOCK/CRYPTO kind)
- `users/{uid}/alerts/{alertId}` — PriceAlert (max 4 per symbol, direction ABOVE/BELOW, once-per-day fire)

## CI/CD (Bitrise — bitrise.yml)
- `feature/*`, `bugfix/*`, `develop` → debug APK build
- `release/*`, `hotfix/*` → signed AAB + universal APK
- `master` → Google Play internal track publish (requires service account)

## Shipped Features (v1.0.2)
- Stock search + full dashboard: price, OHLC chart, Greeks, IV smile, OI by strike, sentiment meter, rule-based signal, AI prediction chart (Claude), AI strategy card (Claude)
- Crypto dashboard: price, news, 24h movers, rule-based signal
- Watchlist (add/remove, Firestore-backed, real-time auth-aware listeners)
- Price alerts (create/toggle/delete, local notifications, once-per-day enforcement)
- News: broad-market, per-symbol stock/crypto, local city-based
- Macro overviews: popular tickers/pairs, top movers, earnings/events
- Admin: user management (list, promote/demote roles)
- Auth: Google Sign-In + Email/Password + ToS gate
- Theme picker (Dark/Light/System)
- First-launch spotlight tutorial wizard (SearchScreen)
- Firebase Analytics + Crashlytics telemetry throughout
- Monochrome/themed launcher icon (Android 13+)

## Known Gaps / Next Features
- **WorldMapScreen** — exists but NOT wired into nav; dead code or WIP
- **Earnings calendar screen** — EventCard component exists; no dedicated screen or full data flow
- **M&A merger news** — MergerCard component exists; Firestore/news data flow not integrated
- **Portfolio tracking** — no holdings, positions, or P&L
- **Multi-timeframe charts** — no MA/RSI/Bollinger bands, no drawing tools, no timeframe switcher
- **Push notifications (FCM)** — foreground local only; no FCM background alerts
- **Options P&L simulator** — no payoff diagram or breakeven visualization
- **Watchlist folders/groups** — flat list only; no sector/risk grouping
- **Greeks-based alerts** — price-only today; no gamma/vega/theta thresholds
- **Broker integration** — no order execution or account linking
- **Compose UI tests (androidTest)** — no androidTest folder exists
- **ViewModel unit tests** — none today; domain + utils only (8 test files)
- **Cache eviction** — PredictionEngine in-memory cache unbounded
- **Network retry logic** — fail-fast today, no exponential backoff

## Test Coverage
8 unit test files: SignalEngineTest, SentimentAnalyzerTest, AppUserTest, OptionsContractTest, CryptoRepositoryTest, SymbolDetectorTest, NumbersTest, TutorialCardPositionTest
No instrumented (androidTest) tests. No ViewModel tests.

Run unit tests: `./gradlew :app:testDebugUnitTest`

**Why:** Focus has been on shipping MVP features; ViewModel/UI test infrastructure is next priority.
**How to apply:** API keys always server-side via Cloud Functions. Follow MVVM/Compose patterns. Target API 26+ / Java 17.
