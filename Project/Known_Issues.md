# Known Issues

Verified defects, risks and placeholders on `main` (`f65e19f`), each checked against the code on 2026-10-02.

---

## Overview

This note lists what is wrong or misleading **today**. When a fix merges, move the row to **Resolved** with the PR number instead of deleting it. Ideas for fixing these live in the backlog table in [[Tickets/README|Tickets]].

Severity: **High** = security or data risk · **Medium** = a visible bug or a misleading screen · **Low** = dead code or developer friction.

---

## Open

| # | Issue | Severity | Where | Details |
|---|-------|----------|-------|---------|
| 1 | Firestore rules allow any signed-in user to read and write **every** document | High | `SmartClothingApp/firestore.rules` | `Users`, `Invitations` and a `{document=**}` catch-all all use `allow read, write: if request.auth != null`. Any athlete can read another athlete's profile and health data, and anyone can append to another user's `pendingPermissions`. The Coaching Dashboard shares the project, so this has to be fixed with that team. See [[Firestore_Rules]] |
| 2 | Coach "disable sharing" can never take effect | Medium | `src/actions/userActions.js` | Sign-up writes `disabledCoachList`, but `fetchCoachAccess` reads `CoachListDisabled`. Nothing writes the second name, so `sharingEnabled` is always true. See [[Coach_Access_Permissions]] |
| 3 | Home heart-rate chart is always flat | Medium | `src/screens/HomeScreen/index.jsx` | `HomeScreen` passes `data={defaultData}`, but `HeartRateChart` reads `dataArray`, so it renders its fallback. See [[Home_Dashboard]] |
| 4 | Launching the app from a notification never opens the permissions modal | Medium | `App.js` | `getLastNotificationResponseAsync()` is called without `await`, so the code receives a Promise. Tapping a notification while the app is running works. See [[Push_Notifications_Client]] |
| 5 | Health and Home screens show placeholder data | Medium | Home, Insights, Health, Devices | Activity, steps and calories cards, breathing and ventilation charts, activity rings, HR recovery scatter, resting HR / HRV rows and the device list are **hard-coded**. No wearable ingestion exists on `main`. See [[Home_Dashboard]], [[Insights_Activity_Rings]], [[Health_Data_Screens]], [[Devices_Tab]] |
| 6 | The app and the dashboard read different health collections | Medium | Firestore | The app reads `HeartRateData` / `SleepData`; the dashboard reads `HeartRateDataHC` / `SleepDataHC` / `VO2MaxDataHX` with different field names. Nothing in this repo writes any of them. See [[Firestore_Schema]] |
| 7 | CI never fails on snapshot drift | Medium | `.github/workflows/main.yml` | `npm test -- -u` rewrites snapshots on the runner. CI runs on PRs, manual dispatch and pushes to `testing` only, on Node 18. See [[CI_GitHub_Actions]] |
| 8 | Commented-out tests | Low | `__test__/` | Three `projectId` cases in `notifications.unit.test.js` (commit `967eb6d`), the metrics cases in `localStorage.unit.test.js`, and two cases in `ViewSleepData.test.js`. Suites pass without exercising those paths. See [[Testing]] |
| 9 | Settings "DELETE DATA" does nothing | Low | `src/screens/Settings` | `SettingsButton` has no `onPress`. See [[Profile_Settings]] |
| 10 | Dead code | Low | `src/navigation/HomeStack.jsx`, `src/network/`, `src/reducers/deviceReducer.js` | Three navigators are never mounted, the axios client has an empty `baseUrl`, and the `GET_DEVICES` case returns the array instead of state (never dispatched). See [[Navigation]], [[Devices_Tab]], [[Redux_State]] |
| 11 | Android emulator corrupts the Metro bundle over the default network path | Low | Dev setup | Use `npx expo start --offline --localhost`. See [[Android_Emulator_Workflow]] |

---

## Resolved

| # | Issue | Fixed by |
|---|-------|----------|
| — | *(none recorded yet)* | |

---

## Key Files

| File | Why it is here |
|------|----------------|
| `SmartClothingApp/firestore.rules` | #1 |
| `SmartClothingApp/src/actions/userActions.js` | #2, #6 |
| `SmartClothingApp/src/screens/HomeScreen/index.jsx` | #3, #5 |
| `SmartClothingApp/App.js` | #4 |
| `.github/workflows/main.yml` | #7 |

---

## Related Notes

- [[Roadmap]]
- [[Testing]]
- [[Firestore_Rules]]
- [[Tickets/README|Tickets]]
