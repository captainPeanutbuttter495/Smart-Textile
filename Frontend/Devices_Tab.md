# Devices Tab

The `Device` bottom tab (`MyDevices`) and `DeviceDetails`. On `main` this is a static mockup with no device connectivity.

---

## Overview

`MyDevices` renders one `DeviceCard` per entry in `state.device.devicesData` and navigates to `DeviceDetails` with the item id. `devicesData` is a hard-coded array in `src/reducers/deviceReducer.js`:

| id | name | device | image | details |
|----|------|--------|-------|---------|
| 1 | Smart Sock | Smart sock v2.0 & Sensoria Core | `assets/images/sock.jpeg` | Sensoria marketing copy |
| 2 | Smart Shirt | HEXOSKIN PROSHIRT V1 | `assets/images/shirt.jpeg` | Hexoskin marketing copy |

`DeviceDetails` shows the image, model string, and the blurb, plus a Back button.

There is **no** connect, pair, scan, authorize, or sync action anywhere, and no health-platform or Bluetooth dependency in `package.json`.

---

## What exists in code but does nothing

| Item | Location | Note |
|------|----------|------|
| `GET_DEVICES`, `GET_DEVICE`, `ADD_DEVICE`, `UPDATE_DEVICE`, `DELETE_DEVICE` | `src/actions/types.js` | Only `GET_DEVICES` has a reducer case, and it returns `state.devicesData` instead of a state object (would break the slice if dispatched); nothing dispatches it |
| `getDevices()` | `src/actions/deviceActions.js` | Plain action creator for `GET_DEVICES`, unused |
| `savePushTokenToBackend`, `removePushTokenFromBackend` | `src/actions/deviceActions.js` | Duplicate the push-token thunks in `userActions.js`; unused |
| `src/network/NetworkManager.js`, `config.js` | axios singleton with `baseUrl: ""` | Scaffold for a REST API (the Hexoskin API was the original intent); unused |
| Hexoskin API comments | `VentilationChart`, `HeartRateChart` | `//minute_ventilation [mL/min](/api/datatype/36/)`, `//heart_rate [bpm](/api/datatype/19/)` |

---

## History (reference only, do not merge)

Two abandoned branches implemented real device data paths and were never merged; both are on Expo 50 and ~680 commits behind `main`:

| Branch | Platform | Approach |
|--------|----------|----------|
| `origin/galaxy-watch-redux` | Android | `react-native-health-connect`; wrote `Users/{uid}/HeartRateDataHC`, `SleepDataHC` (the shapes the Coaching Dashboard still reads) |
| `origin/ios-dev-carlos` | iOS | Custom Swift `Controller` native module over HealthKit; wrote `HeartRateData`, `SleepData` (the shapes this app reads) |

The lead's decision (Sept 2026) is to build Android Health Connect fresh rather than port the branch: [[T-001_Android_Health_Connect]].

---

## Screen States

| State | Display |
|-------|---------|
| Always | Two static cards |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/screens/MyDevices/index.jsx` | List of `DeviceCard`s |
| `src/screens/DeviceDetails/index.jsx` | Detail view |
| `src/components/AppCards/DeviceCard.jsx` | Card UI |
| `src/reducers/deviceReducer.js` | `dummyDevicesData` |
| `src/actions/deviceActions.js` | Unused helpers |

---

## Tests

| Suite | Covers |
|-------|--------|
| none | The Devices tab is a static list |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Home_Dashboard]]
- [[Health_Data_Screens]]
- [[Firestore_Schema]]
- [[T-001_Android_Health_Connect]]
