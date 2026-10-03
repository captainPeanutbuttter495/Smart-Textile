# Firebase Auth

How the app authenticates athletes with Firebase Authentication (email/password only) and keeps the session alive across restarts.

---

## Overview

The app uses the **Firebase JS SDK v10** (`firebase/auth`), not `@react-native-firebase/auth`. The Auth instance is created once in `firebaseConfig.js` with React Native persistence so Firebase keeps its own session in AsyncStorage:

```js
const auth = initializeAuth(app, {
  persistence: getReactNativePersistence(AsyncStorage),
});
```

Only the **email/password** provider is used. `src/components/GoogleButton/` exists as a UI component but there is no Google sign-in flow wired to it.

---

## Operations

| Operation | Thunk / function | Firebase call | Side effects |
|-----------|-----------------|---------------|--------------|
| Sign up | `startSignupWithEmail(email, password, first, last)` | `createUserWithEmailAndPassword` | Creates `Users/{uid}` doc, stores uid/name/email in AsyncStorage, sets `displayName`, opens `DataCollectModal` |
| Sign in | `startLoginWithEmail(email, password)` | `signInWithEmailAndPassword` | Stores uid/name/email in AsyncStorage, dispatches `startLoadUserData()` |
| Set display name | `startUpdateProfile(first, last)` | `updateProfile(user, {displayName: "First Last"})` | Names are re-derived at login by splitting `displayName` on the first space |
| Password reset email | `startSnedPasswordReserEmail(email)` (sic) | `sendPasswordResetEmail` | Plain async function, not a thunk; no toast on success/failure |
| Reauthenticate | `reauthenticate(currentPassword)` | `EmailAuthProvider.credential` + `reauthenticateWithCredential` | Returns `true`/`false`; required by Firebase before email/password change |
| Change password | `updateUserPassword(newPassword)` | `updatePassword` | Called from `ChangePasswordModal` after `reauthenticate` |
| Change email | `updateUserEmail(newEmail)` | `updateEmail` | Updates AsyncStorage + Redux (`updateEmailData`); Firestore `Users/{uid}.email` is **not** updated (the Cloud Function matches invitations by that field) |
| Sign out | `startLogout()` | `signOut` | Clears push token in Firestore, clears every AsyncStorage key, dispatches `logout()` |
| Delete account | `deleteAccount()` | `user.delete()` then `signOut` | Deletes `Users/{uid}`; subcollections are left behind |

All of these live in `src/actions/userActions.js`. Errors are mapped to user-facing strings through `src/utils/firebaseErrorsMessages.js` (a second, near-duplicate `firebaseErrorMessages.js` also exists) and shown via `toastError`.

---

## Session model

Two persistence layers coexist and both must agree for the app to consider the athlete logged in:

| Layer | Where | Set by | Read by |
|-------|-------|--------|---------|
| Firebase Auth session | AsyncStorage (managed by the SDK) | Any successful sign-in | `onAuthStateChanged` in `App.js` |
| App identity cache | AsyncStorage keys `uid`, `firstName`, `lastName`, `email` | `storeUID`, `storeFirstName`, ... in the login/signup thunks | `restoreUUID()` on boot |

`AppRouter` (`src/navigation/index.js`) chooses `AuthStack` vs `AllNavigationStack` purely on `state.user.uuid`, which is only populated from the app identity cache or a fresh login. A Firebase session without a cached `uid` therefore still shows the sign-in screen.

Dev shortcut: set `uuid` to any non-null string in `userReducer` `initialState` to bypass login (see [[Environment_Setup]]).

---

## Key Files

| File | Purpose |
|------|---------|
| `firebaseConfig.js` | `initializeApp`, `initializeAuth` with RN persistence, `getFirestore`; exports `app`, `auth`, `database` |
| `src/actions/userActions.js` | Every Auth call listed above |
| `src/utils/localStorage.js` | AsyncStorage helpers for the identity cache |
| `src/utils/firebaseErrorsMessages.js` | `error.code` to message map |
| `src/screens/SigninScreen`, `SignupScreen`, `ForgotPasswordScreen` | Auth UI |
| `src/components/ChangePasswordModal`, `UpdateEmailModal`, `DeleteAccountModal` | Settings-tab account actions |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__unit-test__/__actions-test__/userActions.unit.test.js` | Auth thunks with `firebase/auth` mocked |
| `__test__/__integration-test__/userActions.integration.test.js` | Login / logout flows with mocked Firebase |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Authentication]]
- [[Firestore_Schema]]
- [[Firebase_Setup]]
- [[Redux_State]]
