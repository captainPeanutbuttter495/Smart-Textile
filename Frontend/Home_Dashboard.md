# Home Dashboard

The Home tab (`HomeScreen`): what it renders, which parts are backed by real data, and which are hard-coded.

---

## Overview

`src/screens/HomeScreen/index.jsx` is the landing tab after login. It shows a greeting, the Daily Insights ring strip, three "Today Status" cards, and three line charts. **Only the greeting is driven by real data.** Everything else is a placeholder that predates any wearable integration.

---

## Sections

| Section | Component | Data source | Status |
|---------|-----------|-------------|--------|
| Greeting | `Text` | `state.user.firstName` | Real |
| Daily Insights | `DailyInsights` -> 7 x `ActivityRings` (Skia) | `state.app.activityRingsData` | Zeros by default; only `updateActivityRings()` (random values, triggered from the Insights screen button) changes them |
| Activity card | `ActivityCard` | literal `"8H 25Min"` | **Hard-coded** |
| Steps card | `ActivityCard` | literal `"8000"` | **Hard-coded** |
| Calories card | `ActivityCard` | literal `"2000 Kcal"` | **Hard-coded** |
| Breath Rate chart | `BreathingRateChart` | ~150-value array inside the component | **Hard-coded**; respiratory rate in breaths/min, a Hexoskin-garment metric |
| Ventilation Rate chart | `VentilationChart` | ~250-value array inside the component; comment cites Hexoskin `/api/datatype/36/` (minute ventilation, mL/min) | **Hard-coded**; Hexoskin-only metric, no watch provides it |
| Heartbeat Rate chart | `HeartRateChart` | `HomeScreen` passes `data={defaultData}` but the component reads `dataArray` | **Bug**: prop mismatch, chart always renders the fallback `[1]` (a flat line); the 60-value `defaultData` array is dead code |
| Metrics modal | `DataCollectModal` | `state.app.userMetricsDataModalVisible` | Real; opened after sign-up or from Profile |

All three charts are `react-native-chart-kit` `LineChart`s with the same dark-green gradient config and no axis labels or units.

---

## What "real" would look like

The Health tab already reads `Users/{uid}/HeartRateData` and `SleepData` from Firestore ([[Health_Data_Screens]]); the dashboard could reuse `queryHeartRateData` for the heart-rate chart today if the prop name were fixed. Steps / calories / activity minutes have no Firestore collection yet. Breath rate and ventilation only exist as concepts because of the Hexoskin shirt; they should be hidden or removed until a Hexoskin data path exists.

See [[T-001_Android_Health_Connect]] for the ticket that replaces the steps and heart-rate placeholders.

---

## Screen States

| State | Display |
|-------|---------|
| Any | Same static content; no loading or empty states exist |
| Just signed up | `DataCollectModal` overlays the dashboard |
| Pending coach invite | `NotificationPermissionsModal` (mounted in `App.js`, not here) overlays the dashboard |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/screens/HomeScreen/index.jsx` | Layout, hard-coded card values, `defaultData` |
| `src/components/DailyInsights/DailyInsights.jsx` | Ring strip + "View" button to `Insights` |
| `src/components/AppCards/ActivityCard.jsx` | Icon + title + value card |
| `src/components/visualizations/BreathingRateChart/index.js` | Hard-coded breathing series |
| `src/components/visualizations/VentilationChart/index.js` | Hard-coded ventilation series |
| `src/components/visualizations/HeartRateChart/index.js` | `dataArray` prop -> `entry.heartRate` |
| `src/components/DataCollectModal/index.jsx` | Metrics form |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__integration-test__/__main-tab-test__/__main-navigation-test/HomeToProfile.test.js` | Home -> Profile navigation |
| `__test__/__integration-test__/__main-tab-test__/__main-navigation-test/HomeToSettings.test.js` | Home -> Settings navigation |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Health_Data_Screens]]
- [[Insights_Activity_Rings]]
- [[Devices_Tab]]
- [[Profile_Settings]]
- [[Navigation]]
