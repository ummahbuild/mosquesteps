# MosqueSteps — Feature integration deep dive

**Audience:** Engineers porting or re-implementing MosqueSteps (native, backend-for-frontend, or QA automation).  
**Companion:** [`mobile-dev.md`](../mobile-dev.md) (routes, APIs, storage keys, native checklist). **This document** explains **logic, settings fields, stats math, and user flows** in depth.

**Canonical product URL:** [https://mosquesteps.com](https://mosquesteps.com) (same host as [https://MosqueSteps.com](https://MosqueSteps.com)).

---

## 0. Mental model

1. **Walk history** (`WalkEntry[]`) — persisted walks (steps, distance, prayer, mosque, time). Drives badges, streaks, totals, exports.
2. **User settings** (`UserSettings`) — single merged JSON blob; **autosaved** from Settings UI with **~400 ms debounce** after any change.
3. **Prayer log** (`DayLog[]`) — *separate* manual attendance / transport log per calendar day; **not** the same as walk history.
4. **Goals** (`WalkingGoals`) — user targets for Stats progress bars; persisted independently.
5. **Notifications** — (a) **system** permission + scheduled reminders in `localStorage`, (b) **in-app** feed in `mosquesteps_notifications`.

---

## 1. Persistence index (logic-relevant keys)

| Key | Type | Written by | Purpose |
|-----|------|-------------|---------|
| `mosquesteps_settings` | JSON `UserSettings` | `saveSettings` / Settings debounce | All configurable behaviour |
| `mosquesteps_history` | JSON `WalkEntry[]` | `addWalkEntry`, `deleteWalkEntry` | Walk log |
| `mosquesteps_saved_mosques` | JSON `SavedMosque[]` | Mosque finder / primary | Saved + primary + favorites |
| `mosquesteps_goals` | JSON `WalkingGoals` | Stats page `saveGoals` | Daily steps / weekly walks / monthly distance targets |
| `mosquesteps_prayer_log` | JSON `DayLog[]` (max 90 days) | Stats Daily Prayer Log, `markPrayerWalked` | Per-day prayer prayed + go/return transport |
| `mosquesteps_notifications` | JSON `AppNotification[]` (max 200) | App events, seed | In-app inbox |
| `mosquesteps_notification_settings` | JSON toggles | Notifications page | Category on/off for feed (not OS permission) |
| `mosquesteps_badges` | JSON `Record<badgeId, ISO earned date>` | `getBadges` when newly earned | First-earned timestamps |
| `mosquesteps_theme` | `"light"` \| `"dark"` \| `"system"` | `useTheme` | Appearance |
| `mosquesteps_locale` | `"en"` \| `"ar"` \| `"ur"` \| `"ms"` \| `"tr"` | `setLocale` in `i18n.ts` | UI strings (partial coverage in app) |
| `mosquesteps_onboarded` | `"true"` | Onboarding complete | Gate main app |
| `mosquesteps_onboarded_date` | ISO string | First completion | Stats “days since onboarding” |
| `mosquesteps_prayer_cache` | Cached prayer payload | `fetchPrayerTimes` | 6 h TTL |
| `mosquesteps_ip_geo` | IP geolocation | `getIPGeolocation` | 24 h TTL |
| `mosquesteps_mosque_cache` | Overpass cache | `offline-cache` | 24 h TTL |
| `mosquesteps_route_cache` | Route cache | `setCachedRoute` | 7 d TTL |
| `mosquesteps_scheduled_reminders` | `ScheduledReminder[]` | Prayer scheduling | Browser notification schedule |
| `mosquesteps_mapbox_token` | string | Settings (advanced) | Optional Mapbox directions |
| `mosquesteps_routing_provider` | `"osrm"` \| `"mapbox"` | Settings | Routing preference |
| `mosquesteps_search_cache_*` | sessionStorage | `searchNearbyMosques` | Session mosque results (clear between tests) |
| `mosquesteps_active_walk` | `"active"` | ActiveWalk while tracking | Session flag for “walk in progress” |

---

## 2. `UserSettings` — every field (validation & effects)

All values pass through `saveSettings` in `walking-history.ts`, which **merges** onto existing settings and **clamps/sanitizes** (see `DEFAULT_SETTINGS` for baselines).

### 2.1 Location & time

| Field | Type | Validation / notes | Downstream effect |
|-------|------|--------------------|-------------------|
| `cityName` | string? | Sanitized: strip `<>` , trim, max 500 chars | Display name for prayer city |
| `cityLat`, `cityLng` | number? | Clamped to valid WGS84 | Aladhan API `latitude` / `longitude` |
| `cityTimezone` | string? | IANA, from TimeAPI or onboarding | `fetchPrayerTimes`, `getNowInTimezone`, “today” / `isPast` |
| `homeAddress` | string? | Sanitized string | Display only |
| `homeLat`, `homeLng` | number? | Clamped | **Dashboard** OSRM route “home → mosque” (only when both + mosque coords set). **Saved mosque** `distanceKm` recomputed via `recomputeMosqueDistancesFromHome` |

### 2.2 Mosque selection (primary)

| Field | Notes |
|-------|--------|
| `selectedMosqueName` | Default `"My Mosque"`; sanitized |
| `selectedMosqueDistance` | km, 0–100; used if no route |
| `selectedMosqueLat`, `selectedMosqueLng` | Required for maps / walk / leave-by from route |

`setPrimaryMosque` in storage copies primary into these fields.

### 2.3 Walking physics

| Field | Type | Validation | Effect |
|-------|------|--------------|--------|
| `walkingSpeed` | km/h | 0.5–20, default 5 | `estimateWalkingTime`, leave-by minutes, Settings “~X min walk” uses `ceil(distance / (speed/60))` i.e. minutes = distance / (speed/60)? **Code:** `Math.ceil(selectedMosqueDistance / (walkingSpeed / 60))` → distance ÷ (speed/60) = **distance × 60 / speed** (minutes) |
| `strideLength` | meters | 0.3–2.0, default ~0.75 in UI | Stride / advanced metrics |

### 2.4 Units & display

| Field | Values | Effect |
|-------|--------|--------|
| `distanceUnit` | `"km"` \| `"mi"` | Stats, History, exports formatting |
| `speedUnit` | `"kmh"` \| `"mph"` | Stats average speed display |
| `smallDistanceUnit` | `"m"` \| `"ft"` | Turn-by-turn small distances |
| `timeFormat` | `"12h"` \| `"24h"` | `formatTime` in `regional-defaults.ts` |

`getRegionalDefaults(cityName, timezone)` suggests defaults from **timezone prefix** and city string (12h countries, imperial countries, feet countries). Applied when picking a city **only if** `timeFormat` / `smallDistanceUnit` were unset logic in Settings (see `selectCitySuggestion`: spreads defaults when `!prev.timeFormat` etc.).

### 2.5 Prayer preferences

| Field | Purpose |
|-------|---------|
| `prayerPreferences` | string[] — which obligatory prayers user cares about (onboarding) |
| `optionalPrayers` | e.g. Taraweeh, Jumuah — voluntary |
| `ramadanMode` | boolean — UI / optional prayer surfacing |
| `prayerCalculationMethod` | key into `PRAYER_CALCULATION_METHODS` (default **`ISNA`**) | Maps to Aladhan `method` integer |
| `prayerMosques` | `Record<prayerName, mosqueId>` | Per-prayer mosque choice in UI |

### 2.6 Notification-related (settings blob)

| Field | Default | Range / notes |
|-------|---------|----------------|
| `notifyLeaveByEnabled` | true | Master for “leave for mosque” scheduling |
| `notifyMinutesBefore` | 5 in UI copy; **saved** clamp 0–120 in `saveSettings` | Minutes **before** computed leave-by time to fire reminder |
| `notifyPrayerTimeEnabled` | true | Remind X min before **adhan** time |
| `notifyMinutesBeforePrayer` | 10 | Slider 0–30 step 5 in UI |

**Per-prayer notification toggles** (Settings): dynamic keys `prayer_fajr`, `prayer_dhuhr`, `prayer_asr`, `prayer_maghrib`, `prayer_isha` — stored on the same `settings` object; **`!== false`** means enabled. These are **not** all declared on the `UserSettings` TypeScript interface but are persisted in JSON if present.

### 2.7 Profile / health

| Field | Validation |
|-------|------------|
| `age` | AGE_MIN–AGE_MAX (5–120) |
| `gender` | `"male"` \| `"female"` \| `""` |
| `bodyWeightKg` | BODY_WEIGHT_KG_MIN–MAX |
| `advancedMetricsMode` | boolean — gates weight for calories / future metrics |

---

## 3. Settings page — UI sections & behaviour

**Autosave:** Any `setSettings` triggers `useEffect` → **400 ms debounce** → `saveSettings(settings)` + brief “saved” UI.

### 3.1 Appearance

- **Theme:** `useTheme()` → `localStorage.mosquesteps_theme`; applies `document.documentElement.classList` for `dark`.
- Values: `light` | `dark` | `system`. System listens to `prefers-color-scheme`.

### 3.2 Measurement units

- Distance km/mi, speed kmh/mph, time 12h/24h, small m/ft — each button sets one field on `settings`.

### 3.3 Location for prayer times

- **Use current location:** `navigator.geolocation` → reverse Nominatim → `fetchTimezone` → updates city + tz; toast.
- **City search:** Debounced **300 ms**, Nominatim suggestions; user **must** select from list (or Enter picks first). `cityValidated` prevents free-text apply.
- **Set / Search:** `handleCitySearch` enforces suggestion selection; merges `getRegionalDefaults` for unset unit fields.

### 3.4 Prayer calculation method

- Radio list from `PRAYER_CALCULATION_METHODS`; stores **string key** (e.g. `MWL`).  
- **API:** `fetchPrayerTimes(..., methodId)` uses `PRAYER_CALCULATION_METHODS[key].id` (default method id **2** = ISNA if missing).

### 3.5 Prayer notifications (compound section)

1. **Browser permission:** separate from settings — `requestNotificationPermission`.
2. **Departure reminder** switch → `notifyLeaveByEnabled`.
3. **Slider** → `notifyMinutesBefore` (0–30 step 5 in UI; server-side clamp to 0–120 on save).
4. **Prayer-time reminder** → `notifyPrayerTimeEnabled` + `notifyMinutesBeforePrayer`.
5. **Per-prayer switches** → `prayer_*` keys.

### 3.6 Favorite mosques

- Uses `getSavedMosques` / `toggleFavoriteMosque` / `reorderFavoriteMosque`.
- **Favorites** drive priority for prayer dropdowns (copy in UI).

### 3.7 Home address (`HomeAddressSearch`)

- **Rule:** User **cannot** save free-typed text; must pick Nominatim row. Blur without selection → error.
- On select: `saveSettings` with `homeAddress`, `homeLat`, `homeLng` then **`recomputeMosqueDistancesFromHome()`** so saved mosque `distanceKm` and primary distance stay consistent.

### 3.8 Language

- `useLocale` → `mosquesteps_locale`. **Not every string** uses `t()` yet; Settings exposes locale picker when implemented.

### 3.9 Walking speed & stride

- Sliders/inputs update `walkingSpeed`, `strideLength`.
- **Benchmark wizard:** manual mode computes stride cm from distance (m) / steps, clamp stride **30–150 cm**; applies `strideLength` in meters (`strideCm/100`). Timed mode also derives speed from distance (m) and elapsed seconds: `speedKmh = clamp((dist/100/sec)*3600, 1, 15)` — note dist input treated as **meters** in formula comment path (verify against `applyTimed` in `Settings.tsx` when porting).

### 3.10 Advanced routing

- **Mapbox token** → `mosquesteps_mapbox_token`; provider `mosquesteps_routing_provider`.  
- Logic in `routing.ts`: Mapbox only if token length > 10; else OSRM; fallback chain.

---

## 4. `getWalkingStats()` — formulas (source: `walking-history.ts`)

| Output | Formula |
|--------|---------|
| `totalSteps` | Sum of `entry.steps` |
| `totalDistance` | Sum of `entry.distanceKm` |
| `totalHasanat` | Sum of `entry.hasanat` |
| `totalWalks` | `history.length` |
| `walksByPrayer` | Count by `entry.prayer` string |
| `currentStreak` | Count consecutive **local calendar days** (from device `Date`) starting **today** backward where each day has ≥1 walk |
| `longestStreak` | Longest run of consecutive local days with ≥1 walk |
| `ramadanWalks` | Count walks where `Intl` Islamic calendar month == 9 |
| `fridayWalks` | `getDay() === 5` |
| `fajrThisWeek` | Fajr walks with `date >= now - 7 days` |
| `jumuahStreak` | Consecutive **weeks** (Sunday week start) where user logged **Jumuah** on a Friday |
| `bestObligatoryPrayersInOneDay` | For each local day, count distinct prayers in `{Fajr,Dhuhr,Asr,Maghrib,Isha}` present in walks that day; take **max** |

**Streak “today” key:** `toLocalDateKey(iso)` uses `Date(iso).getFullYear/Month/Date` — **device local**, not `cityTimezone`. Document this if you add timezone-aware streaks later.

---

## 5. Prayer times & leave-by (logic)

### 5.1 `fetchPrayerTimes(lat, lng, dateOverride?, timezone?, methodId?)`

- Validates coordinates; builds cache key with rounded lat/lng + **calendar date in `timezone`** + method.
- **Aladhan:** `GET /v1/timings/{dd-mm-yyyy}?latitude=&longitude=&method=`.
- **Per prayer `isPast`:** if request day is “today” in that timezone, compare prayer minutes to “now” in that timezone.
- **`isNextDay`:** true when **all** of today’s prayers are already past — UI may show tomorrow.

### 5.2 Leave-by

```text
leaveByMinutes = prayerStartMinutes - walkingMinutes - 5
```

Fixed **5 minute** buffer in `calculateLeaveByTime` (not currently tied to `notifyMinutesBefore`). Wrap with `+24h` if negative.

### 5.3 Estimates

- `estimateSteps(km)` = `round(km * 1333)`
- `estimateWalkingTime(km, speedKmh)` = `round((km / speedKmh) * 60)` minutes
- `calculateHasanat(steps)` = `steps * 2`

---

## 6. Stats page (`/stats`) — metrics & charts

**Data sources:** `getWalkingStats()`, `getWalkHistory()`, `getSettings()`, `getGoals()`, `getOnboardingDate()`, `getStepRecommendation` / `getHealthAssessment`, `getDayLog` / `getRecentLogs` (prayer log).

### 6.1 Goals (`WalkingGoals`)

- Defaults: `dailySteps: 5000`, `weeklyWalks: 10`, `monthlyDistance: 20` (km in storage).
- **Daily steps (today):** sum `entry.steps` where `entry.date` **starts with** local `YYYY-MM-DD` string built from **today** (browser local midnight boundary via `startsWith` on ISO string — **caveat:** timezone).
- **Weekly walks:** count walks with `date >= start of current week (Sunday)` in local JS `Date`.
- **Monthly distance:** sum `distanceKm` for walks in current calendar month.
- **Progress %:** `min(100, current/target)`; monthly goal display converts current km to mi for bar if imperial.

### 6.2 Key totals cards

- Total steps, hasanat, distance, walking time — from stats / sum of `walkingTimeMin`.

### 6.3 Derived “insights”

- **Avg speed:** only walks with `walkingTimeMin > 0`; per walk implied speed = `distanceKm / (walkingTimeMin/60)` km/h; average across those walks.
- **Avg steps / distance / duration:** simple arithmetic mean over **all** walks (including zero-duration if any).
- **Total calories:** `round(totalSteps * 0.04)` (heuristic).
- **Walks per day:** `totalWalks / max(1, daysSinceOnboarding)`; `daysSinceOnboarding` = ceil from onboarding ISO to now.

### 6.4 Health band

- `getStepRecommendation(age, gender)` returns recommended daily steps + copy.
- **avgDailySteps:** sum steps in walks from last `min(30, daysSinceOnboarding)` days ÷ that many days.
- `getHealthAssessment(avgDailySteps, recommendation)` → qualitative band for UI.

### 6.5 Charts / heatmaps

- **Weekly bar data:** last 7 local days; days before onboarding get `steps: -1` to grey out.
- **Monthly:** last 4 weeks buckets.
- **Frequency heatmap:** up to 30 days back, walks per local day.

### 6.6 Daily Prayer Log (separate subsystem)

- Storage: `mosquesteps_prayer_log`.
- **Day model:** For date `YYYY-MM-DD`, five rows Fajr→Isha with `prayed`, `goMethod`, `returnMethod` (`TransportMode`).
- **Transport modes:** `walked`, `car`, `taxi`, `bus`, `bike`, `other`, `""`.
- **Nav:** day offset 0–30; **cannot** go future.
- **Auto-link:** `markPrayerWalked(prayer)` sets `prayed: true`, `goMethod: "walked"` — call when completing a walk for that prayer (integrate in ActiveWalk completion path if mirroring app).

### 6.7 Export (Stats header)

- Same pipeline as History: `buildExportReport(history, stats, { limit: 500 })` → JSON / CSV / Markdown download.

---

## 7. History page (`/history`)

- **Tabs:** Walk log list vs charts (weekly steps, prayer distribution).
- **Weekly chart:** Uses `e.date.startsWith(dateStr)` with **UTC date string** from `toISOString().split("T")[0]` — **differs** from Stats weekly logic in edge cases near midnight UTC vs local. Flag for QA when replicating.
- **Delete:** `deleteWalkEntry` → refreshes list + stats.
- **Export:** `buildExportReport` without limit uses full history in export helper default.

---

## 8. Dashboard (`/dashboard`) — flow

1. **Onboarding gate:** if `!hasCompletedOnboarding()` redirect to `/onboarding` (verify in live `Dashboard.tsx`).
2. **Prayer load:** Resolve coordinates **priority** (typical pattern): try GPS / then settings city / IP / fallback Makkah — then `fetchPrayerTimes` with `cityTimezone` and method from `getSettings().prayerCalculationMethod` → map to id.
3. **Per prayer row:** Show time, **leave by** (from walking time to mosque + buffer), **Walk** CTA → `/walk?prayer=`.
4. **Live clock:** Tick every 1s; display in city timezone where implemented.
5. **Reminders:** `clearScheduledReminders` then `schedulePrayerReminder` / `schedulePrayerTimeReminder` based on toggles; `startReminderPolling` when battery policy allows.
6. **Home→mosque route snippet:** If `homeLat/homeLng` and mosque coords → `fetchWalkingRoute` → show distance/duration card.
7. **Smart alerts:** `calculateAllAlerts` with weather optional (`fetchWeather`).
8. **PWA install:** `pwa-install-store` visit counts + `shouldShowInstallPrompt`.

---

## 9. Active walk (`/walk`) — high-level state machine

**Phases (conceptual):** Pre-walk → Tracking (GPS + step counter) → Paused optional → Completion.

- **Pre-walk:** Requires mosque for full route UI; prayer selector; “Start Walking” enables route fetch.
- **Tracking:** OSRM/Mapbox polyline, turn-by-turn via `formatDirection`, off-route if distance to polyline > **100 m** (config in code).
- **Steps:** `StepCounter` class — accelerometer → device motion → GPS distance fallback; see `step-counter.ts`.
- **Session flag:** `sessionStorage.mosquesteps_active_walk = "active"` while walking; removed on stop/complete.
- **Completion:** Persist `addWalkEntry`; hasanat = `calculateHasanat(steps)` or stored value; optional **check-in** if within **100 m** of mosque; **`markPrayerWalked(selectedPrayer)`** updates `mosquesteps_prayer_log` for today (`goMethod: "walked"`); badge side effects via `getBadges` on next stats read.

**Query params:** `?prayer=Fajr` etc. to preselect.

---

## 10. Mosque finder (`/mosques`)

- **Overpass** with retries + alternate servers; concentric radii; dedupe; session cache.
- **Saved mosques:** `saveMosque`, `setPrimaryMosque`, favorites — distances from home when set.
- **Routing:** `fetchWalkingRoute` from user or map centre to selected mosque.

---

## 11. Rewards (`/rewards`)

- **Badges:** `getBadges(getWalkingStats())` mutates `mosquesteps_badges` when new thresholds crossed.
- **Hadiths tab:** Static content + external links.
- **Categories:** `BADGE_CATEGORIES` map.

---

## 12. Notifications inbox (`/notifications`)

- **Types:** see `NotificationType` in `notification-store.ts`.
- **Filters:** by type, read/unread, sort order — implemented in page + tests `filters.test.tsx`.
- **Settings panel:** `getNotificationSettings` / `saveNotificationSettings` — toggles: `prayerReminders`, `walkUpdates`, `streakAlerts`, `badgeAlerts`, `weeklySummary`, `healthTips`.

---

## 13. Export report (`stats-export.ts`)

`buildExportReport(history, stats, { limit? })` returns:

- `summary`: totals + streaks  
- `walksByPrayer` from stats  
- `walksByDayOfWeek`, `walksByMosque`, `walksByMonth` **recomputed** from raw `history` walk-by-walk  
- `recentWalks`: first N entries of history  

CSV export is **per walk row**, not the aggregate report object.

---

## 14. Edge cases checklist (integration)

- [ ] **Empty history:** Stats goals show 0%; streaks 0; divide-by-zero guarded where needed.  
- [ ] **Corrupt JSON** in any key: readers return defaults / empty; no throw.  
- [ ] **Same-day prayer log + walk:** walk adds steps; log tracks attendance separately.  
- [ ] **Timezone:** Prayer “day” uses `cityTimezone` in API layer; streak uses **local** `Date` — known inconsistency.  
- [ ] **Offline:** `navigator.onLine === false` short-circuits **new** route fetch after cache miss (`routing.ts`).  
- [ ] **Notification permission denied:** app remains usable; scheduling no-ops or skips gracefully.

---

## 15. Source file map (for granular reading)

| Concern | Primary files |
|---------|----------------|
| Settings UI + benchmark | `src/pages/Settings.tsx` |
| Stats UI + prayer log UI | `src/pages/Stats.tsx` |
| Walk CRUD + stats + settings validation | `src/lib/walking-history.ts` |
| Prayer log API | `src/lib/prayer-log.ts` |
| Prayer times + methods | `src/lib/prayer-times.ts` |
| Goals | `src/lib/goals.ts` |
| Badges | `src/lib/badges.ts` |
| Notifications + scheduling | `src/lib/notifications.ts`, `src/lib/notification-store.ts` |
| Routing + Mapbox | `src/lib/routing.ts`, `src/lib/offline-cache.ts` |
| Directions copy | `src/lib/directions-utils.ts` |
| Health copy | `src/lib/health-recommendations.ts` |
| Regional defaults | `src/lib/regional-defaults.ts` |
| i18n | `src/lib/i18n.ts`, `src/hooks/use-locale.ts` |
| Dashboard orchestration | `src/pages/Dashboard.tsx` |
| Active walk | `src/pages/ActiveWalk.tsx` |
| Mosque search | `src/lib/mosque-search.ts` |

---

*Generated from the MosqueSteps codebase. When behaviour diverges from this doc, treat the code as authoritative and update this file in the same change.*
