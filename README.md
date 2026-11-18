# NBA Android Challenge

A small single-activity NBA app built for the TGLAB assessment. Browse and sort teams, search players, and open the same paginated team-games sheet from either tab.

## Stack

Kotlin, Jetpack Compose, Material 3, ViewModel/StateFlow, coroutines, Hilt/KSP, Retrofit/OkHttp, kotlinx.serialization, and Paging 3. Tests use JUnit, MockK, coroutine test dispatchers, MockWebServer, and Compose UI testing.

## Architecture

One Gradle module, organized by feature. Each feature uses `components`, `viewmodel`, `state`, and `enums` packages as needed. Types have their own files; API DTOs end in `Dto`.

```text
com.tglab.nba/
  MainActivity.kt
  NbaApplication.kt
  feature/
    home/       # Components, ViewModel, state, sorting
    players/    # Components, ViewModel, search state
    teamgames/  # Shared sheet and ViewModel
  data/
    model/      # Team, Player, Game and search page
    remote/     # Retrofit API, authentication interceptor, DTOs
    paging/     # Cursor-based games PagingSource
    repository/ # One concrete NbaRepository
  di/           # Hilt bindings
  ui/           # Shared components, errors and theme
```

ViewModels coordinate requests and expose immutable state. Composables render state and invoke callbacks. The concrete repository maps DTOs and builds the games Pager. There are no use cases, repository interfaces, additional Gradle modules, Navigation Compose, or database: these would add little value for two tabs and one sheet.

The root owns saveable tab/team selection and list positions. Sorting and search input are restored through `SavedStateHandle`; server data is fetched again after process death. The games ViewModel cancels its previous selection and scopes Paging's in-memory cache to the open sheet. Closing or switching teams clears that cache.

## API setup

The assignment's legacy URLs are replaced by the current [BALLDONTLIE NBA API](https://docs.balldontlie.io/): `https://api.balldontlie.io/v1/`.

1. Create an account and obtain a key at [BALLDONTLIE](https://app.balldontlie.io/).
2. Add this line to the project's existing `local.properties`, preserving `sdk.dir`:

   ```properties
   BALLDONTLIE_API_KEY=your_api_key_here
   ```

3. Sync and rebuild. `local.properties.example` documents the entry without credentials. `local.properties` is ignored by Git.

The key is sent as `Authorization: YOUR_API_KEY` (no Bearer prefix). Missing configuration fails locally; keys and response bodies are not logged. The build embeds the key in `BuildConfig`, so it can be extracted from an APK. This is suitable for a local assessment, not a production secret-storage solution.

Teams, Players and Games are documented as free-tier endpoints. The free tier allows **5 requests per minute**; a paid key is not required, but the limit can interrupt exploration. HTTP 401/403, 429, connectivity and unexpected response failures have explicit UI states. Retries are user initiated.

## Build and run

Use a compatible Android Studio installation, Android SDK 37 and JDK 25 (the Gradle daemon configuration requests Java 25). The project uses Gradle 9.6.0 / AGP 9.5.1 and targets Java/Kotlin bytecode 17. Minimum Android version: API 24.

On Windows:

```powershell
.\gradlew.bat :app:assembleDebug
```

On macOS/Linux use `./gradlew`. Select an emulator/device and run `app` from Android Studio. The debug APK is at `app/build/outputs/apk/debug/app-debug.apk`.

## Tests

```powershell
.\gradlew.bat :app:testDebugUnitTest
.\gradlew.bat :app:lintDebug
# Requires a running emulator or connected device:
.\gradlew.bat :app:connectedDebugAndroidTest
```

Unit tests cover sorting, state transitions, search debounce/cancellation, retries, cursor pagination, duplicate prevention, team switching, and HTTP request/serialization contracts. Instrumented tests cover the sorting dialog, player selection, games pagination, team switching, and sheet dismissal using Close and system Back.

Tests use explicit synthetic fixtures and a local MockWebServer; they do not require credentials or substitute data in the production app. Instrumented tests also save screenshots under the app's external files `screenshots` directory for visual inspection.

## Behavior and trade-offs

- Search trims surrounding whitespace, debounces for 400 ms, cancels outdated requests immediately, and performs no request for blank input. Additional results use a `Load more` button and 100-item pages, reducing automatic requests under the free-tier limit. Players without a team remain visible with a disabled team-selection action.
- Games use Paging 3, 100-item pages, `meta.next_cursor`, controlled prefetch, retry, stable IDs, duplicate filtering, and an end-of-list indicator. No Room or disk cache is used. A refresh starts at the first page because the API exposes forward cursors.
- The PDF mentions both "recent games" and "all games". The app preserves all games for the selected team in API order; it does not impose a season/date filter or claim globally newest-first ordering. Dates and game status accompany the required scores.
- No polling or live-score refresh is implemented. Reopening the sheet starts a fresh request. Scheduled games retain the API's status; absent scores display a dash.
- Live API smoke check (2026-11-18): Teams, player search, games and a subsequent games page returned HTTP 200. All 89 returned teams matched the required DTO fields. This check covered one player query and one team's first two games pages. Credentials and smoke-response reports remain excluded from version control.
- Lifecycle 2.9.4's custom lint registry is incompatible with the current AGP lint runtime: `UnsafeLifecycleWhenUsage` and `RepeatOnLifecycleWrongUsage` are skipped with an explicit warning. The remaining lint checks run normally; the app uses lifecycle-aware Compose collection rather than these APIs.
