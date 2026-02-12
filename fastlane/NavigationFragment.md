# NavigationFragment.kt

> **File**: `app/src/main/java/net/vonforst/evmap/fragment/NavigationFragment.kt`  
> **Purpose**: Displays the in-app navigation screen with route preview, distance, duration, and a button to launch turn-by-turn navigation.

---

## What Is This File?

When a user taps "Navigate" on a charging station, this fragment shows:
- A **map with the route drawn** as a green polyline
- **Distance** (e.g., "12.5 km")
- **Duration** (e.g., "25 min")
- **Start Navigation** button to open Google Maps/other nav apps

---

## Screen Layout

```
┌──────────────────────────────────┐
│           Map View               │
│    📍 ─── green line ──── ⚡    │
│   (you)                (station) │
├──────────────────────────────────┤
│  Distance: 12.5 km              │
│  Duration: 25 min               │
│                                  │
│  [  🧭 Start Navigation  ]      │
└──────────────────────────────────┘
```

---

## How It Works (Step by Step)

```
1. Fragment receives destination coordinates via Safe Args
   (destLat, destLng — the charging station's location)
         │
         ▼
2. Get user's current GPS location
   (requires ACCESS_FINE_LOCATION permission)
         │
         ▼
3. Load Google Maps API key from resources
   Resource name: "google_directions_key"
         │
         ▼
4. Call RouteService.getRoute(origin, destination, apiKey)
   (runs on IO dispatcher, non-blocking)
         │
         ▼
5. If route received:
   ├── Draw green polyline on map
   ├── Add "Your Location" marker at origin
   ├── Display distance and duration text
   ├── Animate camera to fit entire route
   └── Enable "Start Navigation" button
         │
   If route is null (both APIs failed):
   └── Show error: "Could not calculate route"
```

---

## Key Functions

### `fetchRoute()` — Main route loading function

This is the core function that orchestrates everything:

```kotlin
private fun fetchRoute(originLat: Double, originLng: Double) {
    // 1. Show loading spinner
    // 2. Get API key
    // 3. Call RouteService.getRoute(...)
    // 4. Display results or error
}
```

### `formatDistance(km: Double): String`

```kotlin
formatDistance(0.5)   → "500 m"
formatDistance(12.3)  → "12.3 km"
formatDistance(100.0) → "100.0 km"
```

### `formatDuration(minutes: Double): String`

```kotlin
formatDuration(13.5)  → "13 min"
formatDuration(90.0)  → "1h 30m"
formatDuration(125.0) → "2h 5m"
```

---

## Navigation Arguments (Safe Args)

The fragment receives these arguments from `MapFragment`:

```kotlin
data class NavigationFragmentArgs(
    val destLat: Float,    // Charging station latitude
    val destLng: Float     // Charging station longitude
)
```

---

## API Key Loading

```kotlin
val googleApiKey = try {
    val keyResId = requireContext().resources.getIdentifier(
        "google_directions_key", "string", requireContext().packageName
    )
    if (keyResId != 0) requireContext().getString(keyResId) else null
} catch (e: Exception) { null }
```

This loads the `google_directions_key` string resource that's injected by `build.gradle.kts` from `gradle.properties`.

---

## Route Display

The route is drawn as a **green polyline** on the MapLibre map:

```kotlin
map.addPolyline(
    PolylineOptions()
        .addAll(polylinePoints)         // List of LatLng points
        .color(Color.parseColor("#4CAF50"))  // Material Green
        .width(5f)                      // Line thickness
)
```

The camera is animated to show the entire route:

```kotlin
val boundsBuilder = LatLngBounds.Builder()
polylinePoints.forEach { boundsBuilder.include(it) }
map.animateCamera(CameraUpdateFactory.newLatLngBounds(bounds, 100))
```

---

## Debug Logging

The fragment logs extensively with tag `NavigationFrag`:

```
NavigationFrag: ============ NAVIGATION FRAGMENT ============
NavigationFrag: Location permission: GRANTED
NavigationFrag: GPS Location: (17.3850, 78.4867)
NavigationFrag: google_directions_key resource ID: 2131034122
NavigationFrag: Google API key found (length=39): AIzaSyBxx...
NavigationFrag: Calling RouteService.getRoute()...
NavigationFrag: RouteService returned in 1234ms
NavigationFrag: ✅ Route received: 12.50 km, 25.3 min, 156 points
NavigationFrag: Drawing polyline with 156 points on map
NavigationFrag: Camera animated to fit route bounds
```

---

## How It Connects to Other Files

```
NavigationFragment.kt
    │
    ├──◀ MapFragment.kt     — User taps "Navigate" → opens this fragment
    │                          with destination coordinates
    │
    ├──▶ RouteService.kt     — Calls getRoute() to fetch the driving route
    │
    ├──▶ build.gradle.kts    — API key comes from gradle.properties
    │
    └──▶ MapsActivity.kt     — "Start Navigation" button opens external
                                navigation app (Google Maps, etc.)
```

---

## Key Design Decisions

1. **In-app preview first**: Shows the route inside the app before launching external navigation, so users can see distance/duration without leaving the app.

2. **Graceful error handling**: If both routing APIs fail, shows a user-friendly error message instead of crashing.

3. **Binding null check**: After the async route fetch, checks `if (_binding == null)` to handle the case where the user left the screen while the route was loading.

4. **IO dispatcher**: Route fetching runs on `Dispatchers.IO` to avoid blocking the main thread.
