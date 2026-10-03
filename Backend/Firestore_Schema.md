# Firestore Schema

Every Firestore collection and document shape the Smart Clothing App reads or writes, plus the sibling shapes the Coaching Dashboard depends on in the same project.

---

## Overview

Firestore is the system of record and the only integration point between this app and the Coaching Dashboard. There is no application server; the app talks to Firestore directly with the Firebase JS SDK from `src/actions/userActions.js`.

```
Users/{uid}
  ├── HeartRateData/{id}      read by app        (no writer in this repo)
  ├── SleepData/{id}          read by app        (no writer in this repo)
  ├── HeartRateDataHC/{id}    read by dashboard  (no writer in this repo)
  ├── SleepDataHC/{id}        read by dashboard  (no writer in this repo)
  └── VO2MaxDataHX/{id}       read by dashboard  (no writer in this repo)
Invitations/{id}              written by dashboard, triggers Cloud Function
```

---

## `Users/{uid}`

Document id is the Firebase Auth `uid`. Created by `startSignupWithEmail`; updated with `updateDoc` everywhere else (so fields are only present once something has written them).

| Field | Type | Written by | Notes |
|-------|------|-----------|-------|
| `firstName`, `lastName` | string | signup | Spaces stripped by `startUpdateProfile` |
| `email` | string | signup | **Not updated** by `updateUserEmail`; the Cloud Function matches invitations on this field |
| `createdAt` | Date | signup | |
| `gender` | string | `DataCollectModal` -> `startUpdateUserData` | Values from `src/utils/metrics.js` `Gender` |
| `dob` | Timestamp | same | Modal converts `{seconds, nanoseconds}` back to a Date |
| `height` | number (inches) | same | Slider 0-118 |
| `weight` | number (lbs) | same | Slider 0-800 |
| `sports` | string | same | Values from `metrics.js` `Sports` |
| `expoPushToken` | string or null | `startRegisterPushToken` / `clearPushTokenOnLogout` | Single token per user; overwritten on each device that logs in |
| `pendingPermissions` | array of DocumentReference (`Users/{coachId}`) | dashboard appends; app `arrayRemove`s | Coaches awaiting the athlete's decision |
| `coachList` | array of DocumentReference | app `arrayUnion` / `arrayRemove` | Coaches with access |
| `disabledCoachList` | array | signup (`[]`) | Intended for paused sharing |
| `CoachListDisabled` | array | (nothing in this repo) | `fetchCoachAccess` reads **this** name to compute `sharingEnabled`; the casing mismatch with `disabledCoachList` means the two never meet |
| `role` | string | Coaching Dashboard admin tools | `admin` / `coach` / `athlete`; the app ignores it |

---

## Health subcollections read by this app

Queried in `src/actions/userActions.js:866-929`. Both return `[]` on any error.

### `Users/{uid}/HeartRateData/{id}`

| Field | Type | Used by |
|-------|------|---------|
| `date` | Timestamp | `where("date", ">=", start)`, `where("date", "<=", end)` |
| `heartRate` | number | `ViewHeartRateData` min/max + `HeartRateChart` |

### `Users/{uid}/SleepData/{id}`

| Field | Type | Used by |
|-------|------|---------|
| `startDate` | Timestamp | `where`, `orderBy("startDate", "asc")` |
| `endDate` | Timestamp | client-side filter `endDate` within range |
| `sleepValue` | `"In Bed"`, `"Awake"`, `"Core"`, `"Rem"`, `"Deep"` | `ViewSleepData.parseSleepData` and `getPhaseDuration` |

These labels are Apple HealthKit sleep-analysis names; the shapes were designed by an abandoned iOS HealthKit branch (`origin/ios-dev`, see [[T-002_iOS_Apple_HealthKit]]) and nothing on `main` produces them.

---

## Health subcollections read by the Coaching Dashboard

From `coaching-dashboard/server/server.js` (Insights API, `firebase-admin`). Listed so schema decisions here stay compatible.

### `Users/{uid}/HeartRateDataHC/{id}`

Accepted shapes: `{time, beatsPerMinute}` **or** `{timestamp, bpm}` **or** `{x, y}`. This is the raw `react-native-health-connect` heart-rate sample shape.

### `Users/{uid}/SleepDataHC/{id}`

`{startTime, endTime, stage}` (or `start` / `end` / `sleepStage` / `phase`); `stage` is a **numeric** Health Connect stage code, not a label.

### `Users/{uid}/VO2MaxDataHX/{id}`

Hexoskin-derived VO2 max series. Shape not consumed by this app.

---

## `Invitations/{id}`

Written by the Coaching Dashboard when a coach invites an athlete. The app never reads it; `functions/index.js` reacts to `onCreate`.

| Field | Type | Used by |
|-------|------|---------|
| `athleteEmail` | string | Cloud Function: `Users where email == athleteEmail` |
| `coachId` | string | Push payload |
| `coachFirstName`, `coachLastName` | string | Push title + payload |

---

## Schema divergence (decision needed)

| Concern | App (`main`) | Coaching Dashboard |
|---------|-------------|--------------------|
| Heart rate collection | `HeartRateData` | `HeartRateDataHC` |
| Heart rate fields | `date`, `heartRate` | `time`, `beatsPerMinute` |
| Sleep collection | `SleepData` | `SleepDataHC` |
| Sleep fields | `startDate`, `endDate`, `sleepValue` (string) | `startTime`, `endTime`, `stage` (number) |
| Writer | none (`origin/ios-dev` wrote `HeartRateData` / `SleepData` / `ActivityRingsData`, with a `timestamp`-vs-`date` mismatch on heart rate) | none in either repo (`origin/galaxy-watch-redux` wrote the `*HC` shapes) |

Any wearable ingestion work must pick one contract. The recommendation in [[T-001_Android_Health_Connect]] is to write the dashboard's `*HC` shapes (they are the raw Health Connect record shapes) and update the app's two readers, because the dashboard is the downstream consumer that coaches rely on.

---

## Key Files

| File | Purpose |
|------|---------|
| `src/actions/userActions.js` | All reads/writes (`startSignupWithEmail`, `startUpdateUserData`, `startLoadUserData`, coach-access thunks, `querySleepData`, `queryHeartRateData`) |
| `firestore.rules` | Access rules (see [[Firestore_Rules]]) |
| `firestore.indexes.json` | Empty (`indexes: []`); single-field queries need none |
| `functions/index.js` | `Invitations` trigger |
| `C:\Users\mbg91918\coaching-dashboard\server\server.js` | Dashboard's reads of the `*HC` / `*HX` subcollections |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__unit-test__/__actions-test__/userActions.unit.test.js` | Document shapes the thunks write, against mocked Firestore |
| none | Nothing tests the dashboard-side `*HC` / `*HX` collections |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Firestore_Rules]]
- [[Cloud_Functions]]
- [[Coach_Access_Permissions]]
- [[Health_Data_Screens]]
- [[T-001_Android_Health_Connect]]
