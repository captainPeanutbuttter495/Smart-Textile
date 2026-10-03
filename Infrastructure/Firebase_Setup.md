# Firebase Setup

How the Firebase project is wired into the app and how to deploy the pieces this repo owns (rules, functions).

---

## Overview

One Firebase project is shared by this app and the Coaching Dashboard. The app initialises it client-side from `.env` values in `firebaseConfig.js`; server-side code (`functions/`) uses `firebase-admin` with the default credentials of the deployed function.

---

## Client initialisation (`firebaseConfig.js`)

```js
const app = initializeApp({ apiKey, authDomain, databaseURL, projectId, storageBucket, messagingSenderId, appId, measurementId });
const auth = initializeAuth(app, { persistence: getReactNativePersistence(AsyncStorage) });
const database = getFirestore(app);
export { app, auth, database };
```

- `initializeAuth` (not `getAuth`) is required so the React Native persistence adapter is used.
- Realtime Database, Storage, and Analytics are configured but unused.
- The `@react-native-firebase/*` packages in `package.json` are **not** initialised; the native `google-services.json` / `GoogleService-Info.plist` files are consumed by Expo's build for push (FCM/APNs) only.

---

## Console settings the app depends on

| Area | Setting |
|------|---------|
| Authentication | Email/Password provider enabled |
| Firestore | Native mode; rules from `firestore.rules`; no composite indexes |
| Cloud Functions | `sendInvitationNotification` deployed from `functions/` |
| Cloud Messaging | Android app registered so Expo can relay FCM pushes (`google-services.json`) |
| APNs | Key/certificate uploaded to Expo (EAS credentials), `aps-environment` entitlement in `app.json` |

---

## Deploying

```bash
cd SmartClothingApp
firebase login
firebase use <project-id>
firebase deploy --only firestore:rules      # rules
firebase deploy --only functions            # runs functions/ lint first (firebase.json predeploy)
```

`firebase.json` also lists `storage.rules` (deny-all) and `firestore.indexes.json` (empty).

---

## Local emulation

```bash
cd SmartClothingApp/functions
npm run serve        # firebase emulators:start --only functions
```

The app itself has no emulator wiring (`connectFirestoreEmulator` is not called), so it always talks to the live project.

---

## Key Files

| File | Purpose |
|------|---------|
| `firebaseConfig.js` | Client SDK init |
| `.env` | Config values (see [[Environment_Setup]]) |
| `firebase.json` | CLI targets |
| `firestore.rules`, `storage.rules`, `firestore.indexes.json` | Security + indexes |
| `functions/` | Cloud Functions package |

---

## Related Notes

- [[Environment_Setup]]
- [[Firebase_Auth]]
- [[Firestore_Rules]]
- [[Cloud_Functions]]
