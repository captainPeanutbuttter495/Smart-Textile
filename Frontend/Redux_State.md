# Redux State

Store layout, the four reducers, action naming conventions, and where side effects live.

---

## Overview

`src/store.js` builds a classic Redux 4 store (no Redux Toolkit): `combineReducers({ user, toast, device, app })` with `redux-thunk` and the devtools compose enhancer. `App.js` creates it once at module scope and wraps the tree in `<StoreProvider>`.

Class diagram of the full state shape: [06-Code](../SystemDesign/C4-Diagrams/06-Code.md) section 2.

---

## Reducers

| Slice | File | Holds | Notable cases |
|-------|------|-------|---------------|
| `user` | `src/reducers/userReducer.js` | `uuid`, `firstName`, `lastName`, `email`, `userMetricsData {gender, dob, height, weight, sports}`, `coachAccess[]`, `coachAccessDisabled[]`, `pendingPermissions[]` | `LOGIN_WITH_EMAIL` / `SIGNUP_WITH_EMAIL` set identity; `LOGOUT` nulls it; `UPDATE_USER_METRICS_DATA` replaces metrics wholesale; coach-access array ops |
| `app` | `src/reducers/appReducer.js` | `darkTheme`, `userMetricsDataModalVisible`, `isFromSignUpScreen`, `coachNotificationPermissionsModalVisible`, `measurementSystem`, `activityRingsData` (7 days x 3 rings, all `0`), `heartRateDateRangeData`, `sleepDataDateRangeData` | Modal flags carry `{visibility, isFromSignUpScreen}`; date ranges default to `new Date()` twice (today only) |
| `device` | `src/reducers/deviceReducer.js` | `devicesData`: two hard-coded objects (Sensoria sock, Hexoskin shirt) | `GET_DEVICES` returns the array instead of state (bug, never dispatched) |
| `toast` | `src/reducers/toastReducer.js` | `show`, `message`, `error` | `showToast`, `showErrorToast`, `discardToast` |

Action type strings live in `src/actions/types.js` (`LOGIN_WITH_EMAIL`, `UPDATE_ACTIVITY_RINGS_DATA`, `ToastTypes`, unused `ADD_DEVICE` / `UPDATE_DEVICE` / `DELETE_DEVICE`, ...).

---

## Action conventions

| Pattern | Example | Meaning |
|---------|---------|---------|
| `fooBar()` returns `{type, payload}` | `updateUserMetricsData`, `addToCoachAccess`, `updateHeartRateDateRangeData` | Plain action creator, synchronous |
| `startFooBar()` returns `async (dispatch) => {}` | `startLoginWithEmail`, `startUpdateUserData`, `startAddToCoachAccess`, `startRegisterPushToken` | Thunk that touches Firebase and/or AsyncStorage, then dispatches plain actions |
| Non-thunk async exports | `querySleepData`, `queryHeartRateData`, `fetchUserData` | Called directly from screens with `await`; return data, never dispatch |
| Exceptions to the prefix rule | `fetchPendingPermissions`, `fetchCoachAccess`, `deleteFromCoachAccess`, `removeFromPendingPermissions`, `reauthenticate`, `updateUserPassword`, `updateUserEmail`, `deleteAccount`, `clearPushTokenOnLogout` | Thunks without the `start` prefix |

Nearly every Firebase call lives in `src/actions/userActions.js` (~930 lines). `appActions.js` is UI flags + date ranges + the random `updateActivityRings()`; `deviceActions.js` only has push-token helpers that duplicate `startRegisterPushToken`; `toastActions.js` has `toastInfo` / `toastError` / `discardToast`.

---

## Selectors in use

Screens read state with inline `useSelector` calls; there are no memoised selectors. Frequently read paths:

- `state.user.uuid` (router), `state.user.firstName` (Home greeting), `state.user.userMetricsData` (Profile, DataCollectModal)
- `state.user.pendingPermissions`, `state.user.coachAccess` (permission modals)
- `state.app.activityRingsData` (DailyInsights, ViewInsights), `state.app.heartRateDateRangeData` / `sleepDataDateRangeData` (Health screens, DateToolbar)
- `state.device.devicesData` (Devices tab)
- `state.toast` (`AppToast`)

---

## Persistence boundary

Redux is in-memory only. The `user` slice is rebuilt on boot from AsyncStorage (`restoreUUID`) and Firestore (`startLoadUserData`); thunks are expected to keep Firestore, AsyncStorage, and Redux in sync (see [[Authentication]] key table).

---

## Key Files

| File | Purpose |
|------|---------|
| `src/store.js` | `configureStore()` |
| `src/reducers/*.js` | Four reducers |
| `src/actions/types.js` | Action type constants |
| `src/actions/userActions.js` | Identity, metrics, coach access, push token, health queries |
| `src/actions/appActions.js` | Modal flags, date ranges, activity rings |
| `src/actions/toastActions.js` | Toast helpers |
| `__test__/__unit-test__/__reducers-test__/` | Reducer unit tests (`userReducer`, `toastReducer`) |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__unit-test__/__reducers-test__/userReducer.unit.test.js` | User slice |
| `__test__/__unit-test__/__reducers-test__/toastReducer.unit.test.js` | Toast slice |
| `__test__/__unit-test__/__actions-test__/appActions.unit.test.js` | App action creators |
| `__test__/__unit-test__/__actions-test__/userActions.unit.test.js` | User thunks against mocked Firebase |
| none | `appReducer`, `deviceReducer` have no reducer test |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Authentication]]
- [[Coach_Access_Permissions]]
- [[Health_Data_Screens]]
- [[Testing]]
