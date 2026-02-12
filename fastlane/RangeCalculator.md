# RangeCalculator.kt

> **File**: `app/src/main/java/net/vonforst/evmap/model/RangeCalculator.kt`  
> **Purpose**: Calculates the estimated driving range for an EV based on battery level, vehicle specs, and Indian driving conditions.

---

## What Is This File?

`RangeCalculator` is a **singleton object** (only one instance exists) that answers the question: *"How far can I drive with my current battery?"*

It takes into account real-world factors like:
- 🌡️ Temperature (Indian heat drains battery faster)
- ❄️ AC usage (very common in India = ~10% range reduction)
- 🚗 Driving mode (city/highway/mixed)
- 📉 Real-world vs official range gap (15% haircut on ARAI ratings)

---

## The Three Functions

### 1. `calculateRange()` — Main function

```kotlin
fun calculateRange(
    vehicle: VehicleProfile,      // Which car
    batteryPercent: Double,        // Current charge (0-100)
    acOn: Boolean = true,          // Is AC running? (default: yes)
    drivingMode: String = "mixed", // "city", "highway", or "mixed"
    temperatureC: Double = 35.0    // Outside temperature in °C
): Double                          // Returns: estimated range in km
```

**Example:**
```kotlin
val tataNexton = VehicleProfile.findById("tata_nexon_lr")!!
val range = RangeCalculator.calculateRange(
    vehicle = tataNexton,     // 40.5 kWh battery, 12.5 kWh/100km
    batteryPercent = 80.0,    // 80% charged
    acOn = true,              // AC is on
    drivingMode = "city",     // City driving
    temperatureC = 38.0       // Hot Indian summer
)
// Result: approximately 215 km
```

### 2. `isStationReachable()` — Can I reach a station?

```kotlin
fun isStationReachable(
    vehicle: VehicleProfile,
    batteryPercent: Double,
    distanceKm: Double,        // Distance to the station
    acOn: Boolean = true,
    drivingMode: String = "mixed",
    temperatureC: Double = 35.0
): Boolean                      // true = you can reach it, false = you can't
```

Includes a **10% safety margin** — it only says "yes" if you can reach the station using only 90% of your range.

### 3. `remainingBatteryPercent()` — What battery will I have left?

```kotlin
fun remainingBatteryPercent(
    vehicle: VehicleProfile,
    batteryPercent: Double,
    distanceKm: Double,
    ...
): Double  // Returns: battery % remaining after driving distanceKm
```

**Example:** "If I drive 50 km with 80% battery, I'll arrive with ~65% remaining."

---

## How Range Calculation Works (Step by Step)

```
Step 1: Calculate available energy
┌─────────────────────────────────────┐
│ availableEnergy = 40.5 × (80/100)  │
│                 = 32.4 kWh          │
└─────────────────────────────────────┘

Step 2: Get base efficiency
┌─────────────────────────────────────┐
│ efficiencyPerKm = 12.5 / 100       │
│                 = 0.125 kWh/km      │
└─────────────────────────────────────┘

Step 3: Apply driving mode correction
┌─────────────────────────────────────┐
│ City:    × 0.90 (EVs are MORE      │
│                   efficient in      │
│                   stop-start due    │
│                   to regeneration)  │
│ Highway: × 1.15 (more air drag)    │
│ Mixed:   × 1.00 (no change)        │
└─────────────────────────────────────┘

Step 4: Apply temperature penalty
┌─────────────────────────────────────┐
│ > 40°C: × 1.12 (extreme heat)      │
│ > 35°C: × 1.08 (hot — typical      │
│                  Indian summer)     │
│ > 28°C: × 1.03 (warm)              │
│ < 15°C: × 1.10 (cold — rare in     │
│                  India)             │
│ else:   × 1.00 (ideal 15-28°C)     │
└─────────────────────────────────────┘

Step 5: Apply AC penalty
┌─────────────────────────────────────┐
│ AC On:  × 1.10 (+10% energy use)   │
│ AC Off: × 1.00 (no change)         │
└─────────────────────────────────────┘

Step 6: Apply real-world correction
┌─────────────────────────────────────┐
│ × 1.15 (ARAI ratings are ~15%      │
│          optimistic vs reality)     │
└─────────────────────────────────────┘

Step 7: Calculate final range
┌─────────────────────────────────────┐
│ range = availableEnergy / adjusted  │
│         efficiency                  │
└─────────────────────────────────────┘
```

### Worked Example: Tata Nexon LR, 80%, City, AC On, 38°C

```
Available energy: 40.5 × 0.80 = 32.4 kWh
Base efficiency:  12.5 / 100  = 0.125 kWh/km
After city mode:  0.125 × 0.90 = 0.1125 kWh/km
After 38°C heat:  0.1125 × 1.08 = 0.1215 kWh/km
After AC:         0.1215 × 1.10 = 0.13365 kWh/km
After real-world: 0.13365 × 1.15 = 0.15370 kWh/km

Range = 32.4 / 0.15370 ≈ 210.8 km
```

vs. the official ARAI range of 437 km × 80% = 350 km — that's a **40% difference** from official numbers, which is realistic for Indian conditions!

---

## How It Connects to Other Files

```
RangeCalculator.kt
    │
    ├──▶ VehicleInputFragment.kt  — Calls calculateRange() when user
    │                                adjusts battery slider or picks a vehicle
    │
    ├──▶ VehicleProfile.kt         — Uses vehicle specs (batteryCapacityKwh,
    │                                efficiencyKwhPer100Km) as inputs
    │
    └──▶ MarkerUtils.kt            — The calculated range (in km) is passed
                                     to the map to hide stations beyond range
```

---

## Key Design Decisions

1. **Conservative estimates**: The calculator applies multiple stacking penalties (temperature + AC + real-world correction), which gives a realistic "worst case" range. This prevents the user from getting stranded.

2. **India-optimized**: The temperature ranges and defaults (35°C, AC on) reflect typical Indian driving conditions. The 15% real-world correction accounts for ARAI rating optimism.

3. **Safety margin in reachability**: `isStationReachable()` uses only 90% of calculated range, ensuring the user always has a buffer.

4. **City driving is more efficient for EVs**: Unlike petrol cars, EVs are MORE efficient in city traffic because regenerative braking recaptures energy during stop-start driving.
