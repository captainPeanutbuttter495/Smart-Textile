---
id: T-002
title: "iOS: connect an Apple Watch via HealthKit and sync heart rate, sleep, and steps"
area: iOS / Data
priority: Medium
status: Draft
assignee: —
github_issue: —
created: 2026-09-14
updated: 2026-09-14
---

# T-002: iOS: connect an Apple Watch via HealthKit and sync heart rate, sleep, and steps

## Summary

On `main` the app has no iOS health integration: no HealthKit entitlement, no usage strings in `Info.plist`, and nothing under `src/` that touches HealthKit. An older branch (`origin/ios-dev`, and its build-fix-up twin `origin/ios-dev-carlos`) contains a hand-written Swift HealthKit module and a Firestore upload service that were demoed in March 2024, but the branch is ~690 commits behind, on Expo 50, and its data path never worked end to end (see "Current state" below).

This ticket adds Apple HealthKit as the iOS data source, mirroring T-001 (Android Health Connect). Anything the Apple Watch records lands in the Health app; the app reads heart rate, sleep stages, and daily steps from HealthKit and writes them to the same Firestore collections the Coaching Dashboard already reads (`HeartRateDataHC`, `SleepDataHC`, `StepsDataHC`) with `source: "healthkit"`. The Devices tab gets an "Apple Health" card with Connect / Sync now / Disconnect, and the existing Health tab and Home dashboard display the data with no further changes.

**Depends on T-001 merging first.** The Firestore writers, the re-pointed readers, the `deviceReducer` slice, and the Devices-tab card pattern are all built there; this ticket adds the iOS service and wires it into them.

## Why

- Apple Watch has no cloud API; HealthKit on the phone is the only way to read its data.
- One integration covers every iOS athlete regardless of watch model.
- The EAS development build ([[iOS_EAS_Build]]) is already required for push notifications, so no new tooling.
- Reusing T-001's writers and schema means both platforms produce identical documents and the dashboard needs no changes..

## Current state of iOS and data syncing (investigated 2026-09-14)

Recorded here so nobody re-investigates. All findings are from reading the branches; none of this can be run without a Mac, a physical iPhone, and an Apple Watch.

### Branches

| Branch | Forked from `main` | Behind / ahead | Last commit | Toolchain |
|---|---|---|---|---|
| `origin/ios-dev` | 2024-04-04 (`50d0393`) | 691 / 22 | 2025-03-31 | Expo 50 / RN 0.73.5 |
| `origin/ios-dev-carlos` | same | 691 / 24 | 2025-04-14 | Expo 50 / RN 0.73.5 |
| `origin/galaxy-watch-redux` | 2024-04-22 (`2bcac54`) | 682 / 41 | 2025-09-08 | Expo 50 |
| `main` | — | — | — | Expo 52 / RN 0.76, `newArchEnabled: false` |

`ios-dev-carlos` = `ios-dev` + 2 commits touching only `Podfile.lock`, `package-lock.json`, a `DEVELOPMENT_TEAM` swap in `project.pbxproj`, and an `NSLocalNetworkUsageDescription` in `Info.plist`. It is a local build fix-up, not new functionality.

### What `ios-dev` implements

Native (legacy `NativeModules` bridge, not an Expo Module):
- `ios/Controller.swift` (536 lines) + `ios/Controller.m` export `findHealthData`, `requestAuthorization`, `readHeartRateData`, `readHeartRateVariabilityData`, `readRestingHeartRateData`, `readSleepData`, `readActivityRingsData`. Each takes ISO start/end strings and resolves an array of dictionaries.
- The Xcode group has no `path`, so the **outer** `ios/Controller.swift` is what compiles. `ios/SmartClothingApp/Controller.swift` is a stale 58-line duplicate that is not in the build.
- `SmartClothingApp.entitlements` requests `com.apple.developer.healthkit` **plus** `health-records` (clinical records, needs separate Apple approval) and `background-delivery`; the code uses neither.

JavaScript:
- `src/utils/AppleHealthKit/AppleHealthKitUtils.js` wraps the native calls.
- `src/services/AppleHealthKit/firebaseHealthKitService.js`: `performInitialDataSync()` (last 7 days) and `updateWithLatestData()` (since the newest Firestore doc). Uploads with `addDoc` to `Users/{uid}/HeartRateData`, `SleepData`, `ActivityRingsData`.
- Triggers: `App.js` calls `findHealthData()` and `requestHealthKitAuthorization()` unconditionally at boot; `SignupScreen` dispatches `initialHealthDataSync(true)` so `HomeScreen` runs the initial sync behind a `LoadingOverlay`; `RefreshView` (pull-to-refresh on Home) calls `updateWithLatestData()`.

### Why it is reference-only, not mergeable

1. **Heart-rate field mismatch.** `getHeartRateData` returns `{heartRate, timestamp}` but every reader (`queryHeartRateData` on both branches and `#getLatestDateFromCollection`) filters on `date`. Uploaded heart-rate docs are invisible to the Health tab, and incremental sync always restarts from "now".

2. **Duplicates.** Every upload is `addDoc` with no deterministic id; every pull-to-refresh re-inserts the overlapping window for all three collections

3. **Uploads are not awaited.** `performInitialDataSync` / `updateWithLatestData` call `this.uploadX(...)` without `await`; the loading overlay dismisses before writes finish and errors are swallowed.

4. **No platform guard.** `App.js` calls the native module on every platform; on Android `Controller` is `undefined`.

5. **Stubs.** `queryActivityRingsData` returns `[]`; `getHeartRateVariabilityData` returns `undefined`; `getRestingHeartRateData` maps a non-existent field. None are used.

6. **Wrong schema.** It writes the app's own `HeartRateData` / `SleepData` / `ActivityRingsData` collections. The Coaching Dashboard reads `HeartRateDataHC` / `SleepDataHC`; T-001 standardises on those.

7. **Toolchain gap.** Expo 50 -> 52 and RN 0.73 -> 0.76 across 691 commits; `package.json`, `ios/` (pbxproj, Podfile.lock, AppDelegate), `android/`, and `App.js` all conflict. The merge would be a rewrite.

What *is* worth reusing from it: the list of HealthKit types and units (`count/min`, HRV in `ms`, `HKActivitySummaryQuery` for rings), the `HKCategoryValueSleepAnalysis` -> label mapping (`inBed`, `awake`, `asleepCore`, `asleepDeep`, `asleepREM`, `asleepUnspecified`), and the general shape of "initial backfill + incremental sync since latest doc".

### `galaxy-watch-redux`

Already decided in T-001: reference only, do not port. Nothing found today changes that.

## Scope

Follow T-001's layering exactly: **screen -> `deviceActions` / `userActions` thunk -> `src/services/healthKit.js` -> HealthKit**. The service never imports Redux or Firebase; only `userActions.js` talks to Firestore.

### 1. Dependency and native config
- Add `@kingstinct/react-native-healthkit` (Expo Modules API, ships an Expo config plugin). Verify the release is compatible with Expo SDK 52 / RN 0.76 with `newArchEnabled: false` before anything else; if it is not, fall back to `react-native-health` and note it in the PR.
- `app.json`: register the config plugin; it adds the `com.apple.developer.healthkit` entitlement and `NSHealthShareUsageDescription`. Do **not** request `health-records` or `background-delivery`.
- Run `npx expo prebuild --platform ios` and confirm the `ios/` diff is only the entitlement, `Info.plist` string, and Podfile changes.
- Add a manual mock at `__test__/__mocks__/@kingstinct/react-native-healthkit.js` (same pattern as `expo-notifications`) so Jest never loads the native module.

### 2. HealthKit service (`src/services/healthKit.js`)
Thin wrapper, iOS only, same method names and return shapes as T-001's `src/services/healthConnect.js`:
- `isAvailable()` -> `isHealthDataAvailable()` mapped to `"available" | "unavailable"` (no "needs_update" state on iOS).
- `requestPermissions()` -> request **read** access to `HeartRate`, `SleepAnalysis`, `StepCount`; returns the granted list. HealthKit does not report per-type grant status for reads, so treat a resolved request as "connected".
- `getGrantedPermissions()` -> the list saved locally after the last successful request; `revokeAllPermissions()` -> clear that list (HealthKit has no revoke API; the user revokes in Settings -> Health).
- `readHeartRate(start, end)` -> `[{ time: Date, beatsPerMinute: number }]` (unit `count/min`).
- `readSleep(start, end)` -> `[{ startTime, endTime, stage: number }]` using the numeric Health Connect stage codes T-001 stores, mapped from `HKCategoryValueSleepAnalysis`:

  | HealthKit value | Stored `stage` (Health Connect code) |
  |---|---|
  | `asleepDeep` | `STAGE_TYPE_DEEP` (5) |
  | `asleepCore` | `STAGE_TYPE_LIGHT` (4) |
  | `asleepREM` | `STAGE_TYPE_REM` (6) |
  | `awake` | `STAGE_TYPE_AWAKE` (1) |
  | `inBed` | `STAGE_TYPE_AWAKE_IN_BED` (7) |
  | `asleepUnspecified` | `STAGE_TYPE_SLEEPING` (2) |

  Confirm the numeric constants against `react-native-health-connect` once T-001 lands; the point is that `querySleepData`'s stage -> label mapper from T-001 works unchanged.
- `readSteps(start, end)` -> daily `HKStatisticsCollectionQuery` (cumulative sum) -> `[{ date: "YYYY-MM-DD", count: number }]`.

### 3. Firestore writers
Reuse T-001's `startSyncHealthConnect` writers. Generalise the thunk to `startSyncWearable(service, source, startDate, endDate)` (or add a sibling `startSyncHealthKit` that calls the same three writer helpers) so the document shapes and deterministic ids are shared:

| Collection | Doc shape | Doc id |
|---|---|---|
| `Users/{uid}/HeartRateDataHC` | `{ time, beatsPerMinute, source: "healthkit" }` | `hk_${time.toISOString()}` |
| `Users/{uid}/SleepDataHC` | `{ startTime, endTime, stage, source }` | `hk_${startTime.toISOString()}` |
| `Users/{uid}/StepsDataHC` | `{ date, count, source }` | `hk_${date}` |

Set `lastHealthKitSync: Timestamp` on `Users/{uid}` (sibling of `lastHealthConnectSync`).

### 4. Readers
No change. T-001 already points `queryHeartRateData` / `querySleepData` at the `*HC` collections and maps `stage` to labels.

### 5. Incremental sync on app open
In `App.js`, after `restoreUUID` resolves and only when `Platform.OS === "ios"` and the local granted list is non-empty: read `lastHealthKitSync` (default 30 days ago) and dispatch the sync. Never block the UI; failures log and toast.

### 6. Redux (`src/reducers/deviceReducer.js`)
Add a sibling key to T-001's `healthConnect`:
```js
healthKit: {
  status: "unknown" | "unavailable" | "not_connected" | "connected",
  permissions: [],
  lastSync: null,
  syncing: false,
}
```
Thunks `startCheckHealthKit`, `startConnectHealthKit`, `startDisconnectHealthKit` in `src/actions/deviceActions.js`, same shape as the Health Connect ones.

### 7. UI
- **Devices tab** (`src/screens/MyDevices/index.jsx`): render the "Apple Health" card on iOS and the "Health Connect" card on Android (one `WearableCard` component, two configs). Same status / last-sync / Connect / Sync now / Disconnect layout.
- **Home** and **Health tab**: no changes; verify they show the synced range on an iPhone.

### 8. Tests
- Unit: sleep-value mapper, sample flattening, steps aggregation, reducer.
- Integration: `MyDevices` renders the Apple Health card for each status on iOS (`Platform.OS` mocked).
- `npm test` stays green on Windows/Linux CI (no native module loaded).

## Out of scope
- HRV, resting heart rate, activity rings (the old branch read them; add as a follow-up once the three core metrics work).
- HealthKit background delivery / observer queries (sync only on app open and manual refresh).
- Clinical health records.
- Any change to the Android path or the Coaching Dashboard.
- Merging or cherry-picking from `origin/ios-dev` / `origin/ios-dev-carlos`.

## Acceptance criteria
- [ ] On a physical iPhone paired with an Apple Watch, tapping **Connect** on the Devices tab opens the Health permission sheet; granting shows status "Connected" and a last-sync time.
- [ ] Firestore shows new docs under `HeartRateDataHC`, `SleepDataHC`, `StepsDataHC` with `source: "healthkit"`; re-syncing does not create duplicates.
- [ ] The Coaching Dashboard's athlete view shows the heart-rate and sleep data without dashboard code changes.
- [ ] `ViewHeartRateData`, `ViewSleepData`, and the Home Steps card display the synced range.
- [ ] On an iPad or a device with Health unavailable, the card shows "Unavailable" and nothing crashes.
- [ ] Android behaviour is unchanged; no HealthKit import is reachable outside `Platform.OS === "ios"` paths.
- [ ] `git diff main -- ios/` contains only the entitlement, `Info.plist` usage string, and Podfile changes.
- [ ] `npm test` passes; the new mock lives in `__test__/__mocks__/`.

## Reference material
- Library: https://github.com/kingstinct/react-native-healthkit (Expo config plugin, `queryQuantitySamples`, `queryCategorySamples`, `queryStatisticsForQuantity`).
- Fallback library: https://github.com/agencyenterprise/react-native-health.
- Apple: `HKCategoryValueSleepAnalysis`, `HKQuantityTypeIdentifier.heartRate` / `.stepCount`.
- T-001 (the Android Health Connect ticket) for the writers, readers, reducer, and card pattern this ticket reuses.
- Old prototype, **reference only, do not merge or cherry-pick** (Expo 50, 691 commits behind): `origin/ios-dev` — `ios/Controller.swift`, `ios/Controller.m`, `src/services/AppleHealthKit/*.js`, `src/utils/AppleHealthKit/AppleHealthKitUtils.js`, `src/components/RefreshView/index.jsx`.

