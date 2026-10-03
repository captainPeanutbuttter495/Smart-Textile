# Health Data Screens

The Health tab: `ViewHealthData` hub, the Heart Rate and Sleep detail screens, and the date toolbar that drives their Firestore queries.

---

## Overview

The Health tab is the only part of the app that reads health time-series from Firestore. Each detail screen subscribes to a date range in Redux, calls a non-thunk query function from `userActions.js` when the range changes, and renders the result. Because nothing on `main` writes those collections, a fresh account sees empty charts and `N/A` values.

---

## `ViewHealthData` (route `Health Data`)

- `AppHeader` + a `DateToolbar dateType="single"` (no `dataType`, so it reads a local today/today range and dispatches nothing useful).
- Two `DataButton`s navigate to `HeartRateData` and `SleepData`, passing `previousScreenTitle`.

---

## `ViewHeartRateData` (route `HeartRateData`)

| Element | Source |
|---------|--------|
| `DateToolbar dateType="period" dataType="Heart Rate"` | Dispatches `updateHeartRateDateRange(start, end)` -> `state.app.heartRateDateRangeData` |
| `HeartRateChart dataArray={heartRateData}` | `queryHeartRateData(start, end)` -> `Users/{uid}/HeartRateData` where `date` in range; maps `entry.heartRate` |
| "Heart Rate" row | `min - max BPM` computed from the result, or `N/A` when empty |
| "Resting Heart Rate" row | literal `N/A` (**hard-coded**) |
| "Heart Rate Variability" row | literal `N/A` (**hard-coded**) |
| "Heart Rate Recovery" scatter | `victory-native` `CartesianChart` + `Scatter` over a 5-point literal array (**hard-coded**) |

---

## `ViewSleepData` (route `SleepData`)

| Element | Source |
|---------|--------|
| `DateToolbar dateType="period" dataType="Sleep Data"` | Dispatches `updateSleepDataDateRange` -> `state.app.sleepDataDateRangeData` |
| `querySleepData(start, end)` | `Users/{uid}/SleepData` where `startDate` in range, `orderBy("startDate")`, then client-side `endDate` filter |
| "Time in Bed" / "Time Asleep" | `getPhaseDuration("In Bed")`, `getPhaseDuration("Unknown")` via `calculateTotalDuration` (`src/utils/dateConversions.js`) |
| Sleep stage chart | `victory-native` `CartesianChart` + `Line` over `parseSleepData(result)` |
| Awake / REM / Core / Deep rows | `getPhaseDuration("Awake" | "Rem" | "Core" | "Deep")` in `h min` |
| `DailyMetrics name="Sleep"` | Day-of-week selector strip; purely visual, selection changes nothing |

`parseSleepData` maps each document to `{x, y}`: `x` = cumulative hours since the first record scaled to `0-200`, `y` = `Deep 0`, `Core 60`, `Rem 130`, `Awake 180`. Documents with any other `sleepValue` (including `"In Bed"`) are skipped in the chart but counted in the duration rows.

---

## `DateToolbar`

`src/components/DateToolbar/DateToolbar.jsx` wraps `DatePicker` (single) or `DateRangePicker` (period) built on `react-native-calendars` (`BaseCalendar`). It selects which Redux range to display via `props.dataType` (`"Heart Rate"` or `"Sleep Data"`) and dispatches the matching `update...DateRange` thunk on confirm. Both ranges initialise to `startDate = endDate = new Date()`.

---

## Expected Firestore shapes

| Collection | Fields | Consumer |
|-----------|--------|----------|
| `Users/{uid}/HeartRateData` | `date: Timestamp`, `heartRate: number` | `ViewHeartRateData` |
| `Users/{uid}/SleepData` | `startDate: Timestamp`, `endDate: Timestamp`, `sleepValue: "In Bed" / "Awake" / "Core" / "Rem" / "Deep"` | `ViewSleepData` |

These differ from what the Coaching Dashboard reads; see the divergence table in [[Firestore_Schema]].

---

## Screen States

| State | Display |
|-------|---------|
| No documents in range | Heart: `N/A` + flat chart at `1`; Sleep: `0 hr 0 min` rows + empty chart |
| Query error | `console.error`; function returns `[]`, same as empty |
| Range changed | `useEffect` on `dates.startDate` / `dates.endDate` re-queries |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/screens/ViewHealthData/index.jsx` | Hub with two `DataButton`s |
| `src/screens/ViewHeartRateData/index.jsx` | HR chart, min/max, hard-coded recovery scatter |
| `src/screens/ViewSleepData/index.jsx` | Sleep parsing + chart + phase durations |
| `src/actions/userActions.js:866-929` | `querySleepData`, `queryHeartRateData` |
| `src/components/DateToolbar/*` | Date pickers + Redux range dispatch |
| `src/components/visualizations/HeartRateChart/index.js` | chart-kit line chart |
| `src/utils/dateConversions.js` | `calculateTotalDuration` |
| `__test__/__integration-test__/__main-tab-test__/__main-screen-test__/ViewSleepData.test.js` | Snapshot + render test with mocked Firebase |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__integration-test__/__main-tab-test__/__main-screen-test__/ViewSleepData.test.js` | Sleep screen render + snapshot; two cases are commented out |
| `__test__/__unit-test__/__actions-test__/userActions.unit.test.js` | `queryHeartRateData`, `querySleepData` |
| none | `ViewHeartRateData` has no screen test |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Firestore_Schema]]
- [[Home_Dashboard]]
- [[Redux_State]]
- [[T-001_Android_Health_Connect]]
