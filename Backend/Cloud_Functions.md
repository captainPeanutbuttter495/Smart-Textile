# Cloud Functions

The one Firebase Cloud Function this repository deploys, how it is triggered, and how to run and deploy it.

---

## Overview

`SmartClothingApp/functions/` is a separate Node package (`firebase-functions@5`, `firebase-admin@12`, `expo-server-sdk@3`). It is the only server-side code the app owns. `firebase.json` registers it as the `default` codebase and runs `npm run lint` as a predeploy step.

---

## `sendInvitationNotification`

**Trigger:** `functions.firestore.document("Invitations/{invitationId}").onCreate`

**Flow:**

1. Read `athleteEmail`, `coachId`, `coachFirstName`, `coachLastName` from the new invitation.
2. `db.collection("Users").where("email", "==", athleteEmail).get()`.
3. For every matching user document with an `expoPushToken`, call `sendPushNotification(token, title, body, data)`.
4. `sendPushNotification` builds one Expo message and calls `expo.sendPushNotificationsAsync(messages)`; receipts are logged, never persisted.

**Message:**

| Field | Value |
|-------|-------|
| `title` | `Allow Coach {first} {last} to Track Your Data?` |
| `body` | `Coach wants access to your fitness data for tracking and insights. Allow sharing?` |
| `sound` | `default` |
| `data` | `{ screen: "Home", showPermissionsModal: true, coachId, coachFirstName, coachLastName }` |

The `data` payload is the contract with `App.js` `handleNotificationResponse`, which opens `NotificationPermissionsModal` when `screen === "Home" && showPermissionsModal` and a uid is cached.

**What it does not do:** it does not write `pendingPermissions`; the dashboard is expected to do that. It does not chunk or retry Expo messages, and it does not validate the token with `Expo.isExpoPushToken`.

---

## `helloWorld`

`functions.https.onRequest` returning `"Hello from Firebase!"`. Scaffold stub, unused.

---

## Commands

```bash
cd SmartClothingApp/functions
npm install
npm run lint        # eslint (google config); also runs on deploy
npm run serve       # firebase emulators:start --only functions
npm run shell       # firebase functions:shell
npm run deploy      # firebase deploy --only functions
npm run logs        # firebase functions:log
```

---

## Key Files

| File | Purpose |
|------|---------|
| `functions/index.js` | Both functions |
| `functions/package.json` | Dependencies + scripts |
| `functions/.eslintrc.js` | Lint config enforced at deploy |
| `firebase.json` | Codebase registration + predeploy |

---

## Tests

| Suite | Covers |
|-------|--------|
| none | `functions/` has no test script and no tests |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Push_Notifications_Client]]
- [[Coach_Access_Permissions]]
- [[Firestore_Schema]]
- [[Firebase_Setup]]
