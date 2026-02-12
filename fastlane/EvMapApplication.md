# EvMapApplication.kt

> **File**: `app/src/main/java/net/vonforst/evmap/EvMapApplication.kt`  
> **Purpose**: The Application class — runs once when the app starts. Sets up crash reporting, background workers, and theming.

---

## What Is This File?

`EvMapApplication` is the **first code that runs** when the ChargeX app launches. Before any screen is shown, this class:

1. 🎨 Applies the user's dark/light mode preference
2. 🌐 Sets the app language
3. 🐛 Initializes crash reporting (ACRA)
4. ⏰ Schedules background workers for cache cleanup and data updates

---

## What Happens at Startup

```
App Process Created
         │
         ▼
EvMapApplication.onCreate()
         │
         ├── 1. Load user preferences (dark mode, language)
         │
         ├── 2. Apply night mode (dark/light/system)
         │
         ├── 3. Convert legacy language preference to new format
         │
         ├── 4. Initialize SDK utilities
         │
         ├── 5. Set up ACRA crash reporting (production only)
         │       ├── Report format: JSON
         │       ├── Send via HTTP POST
         │       └── Show dialog asking user to describe the crash
         │
         ├── 6. Schedule CleanupCacheWorker (daily)
         │       └── Removes old cached data when phone is idle
         │
         └── 7. Schedule UpdateFullDownloadWorker (weekly)
                 └── Updates offline charging station data
                     (only on WiFi, when battery is good, phone is idle)
```

---

## Background Workers

### CleanupCacheWorker (runs daily)
```kotlin
PeriodicWorkRequestBuilder<CleanupCacheWorker>(Duration.ofDays(1))
    .setConstraints(
        requiresBatteryNotLow = true,     // Don't run on low battery
        requiresDeviceIdle = true          // Only when phone is not in use
    )
```
Removes expired cache entries from the Room database to keep the app lean.

### UpdateFullDownloadWorker (runs weekly)
```kotlin
PeriodicWorkRequestBuilder<UpdateFullDownloadWorker>(Duration.ofDays(7))
    .setConstraints(
        requiresBatteryNotLow = true,
        requiresNetworkUnmetered = true,  // Only on WiFi
        requiresDeviceIdle = true
    )
```
Updates the offline charging station database. Only runs on WiFi to avoid using mobile data.

---

## Crash Reporting (ACRA)

In production builds (`!BuildConfig.DEBUG`), the app uses ACRA to:
1. Catch unhandled exceptions
2. Show a dialog asking the user what they were doing
3. Send the crash report as JSON to the backend server

```kotlin
initAcra {
    reportFormat = StringFormat.JSON
    httpSender {
        uri = getString(R.string.acra_backend_url)
        httpMethod = HttpSender.Method.POST
    }
    dialog {
        text = getString(R.string.crash_report_text)
        commentPrompt = getString(R.string.crash_report_comment_prompt)
    }
    limiter { enabled = true }  // Don't spam reports
}
```

---

## How It Connects to Other Files

```
EvMapApplication.kt
    │
    ├──▶ PreferenceDataSource.kt     — Reads dark mode and language preferences
    │
    ├──▶ CleanupCacheWorker.kt       — Scheduled background worker for cache
    │
    ├──▶ UpdateFullDownloadWorker.kt  — Scheduled background worker for data updates
    │
    └──▶ ConfigurationUtils.kt       — updateNightMode(), updateAppLocale()
```
