# Authentication

Sign-up, sign-in, password reset, session restore, and logout as experienced from the mobile app.

---

## Overview

`AuthStack` (`src/navigation/AuthStack.jsx`) holds three screens: `SignUp`, `SignIn`, `Forgot`. It is shown whenever `state.user.uuid` is null. Every Firebase call goes through thunks in `src/actions/userActions.js`; see [[Firebase_Auth]] for the Firebase side.

---

## Flows

### Sign in
1. `SigninScreen` collects email + password and dispatches `startLoginWithEmail`.
2. On success the thunk stores `uid`, `firstName`, `lastName`, `email` in AsyncStorage (names come from splitting `user.displayName` on the first space), dispatches `startLoadUserData()` to pull `Users/{uid}` into `state.user.userMetricsData`, then dispatches `loginWithEmail(...)`.
3. `state.user.uuid` becomes non-null and `AppRouter` swaps in `AllNavigationStack`.
4. Errors: `toastError(firebaseErrorsMessages[error.code])`; the thunk re-throws so the screen can stop its spinner.

### Sign up
1. `SignupScreen` collects first/last name, email, password (+ ToS modal) and dispatches `startSignupWithEmail`.
2. Thunk creates the Auth user, writes the initial `Users/{uid}` document, stores identity in AsyncStorage, sets `displayName`, dispatches `userMetricsDataModalVisible(true, true)` and `signupWithEmail(...)`.
3. `DataCollectModal` (mounted on `HomeScreen`) opens with "Skip" wording because `isFromSignUpScreen` is true; Save or Skip calls `startUpdateUserData`.

### Forgot password
`ForgotPasswordScreen` calls `startSnedPasswordReserEmail(email)` directly (it is a plain function, not a thunk). No success/failure feedback is shown.

### Session restore (cold start)
`App.js` waits for `onAuthStateChanged`; if a Firebase user exists **and** `getUID()` returns a cached uid, it dispatches `restoreUUID()` which re-populates `state.user` from AsyncStorage. Metrics come from `getMetrics()` first, Firestore second. See the state machine in [06-Code](../SystemDesign/C4-Diagrams/06-Code.md).

### Logout
`AppHeader` menu "Logout" -> `startLogout()`: clears `expoPushToken` in Firestore, `signOut()`, clears every AsyncStorage key, dispatches `logout()` and a toast. See sequence 5 in [07-Dynamic-Diagrams](../SystemDesign/C4-Diagrams/07-Dynamic-Diagrams.md).

---

## AsyncStorage keys (`src/utils/localStorage.js`)

| Key | Set by | Cleared by |
|-----|--------|-----------|
| `uid` | login, signup | logout, delete account |
| `firstName`, `lastName` | login, signup, `startUpdateProfile` | logout, delete account |
| `email` | login, signup, `updateUserEmail` | logout, delete account |
| `metricsData` | `startUpdateUserData` | logout, delete account |
| `authToken` (Expo push token) | `startRegisterPushToken` | `clearPushTokenOnLogout` |

---

## Screen States

| State | Display |
|-------|---------|
| Not signed in | `AuthStack` (SignIn default) |
| Signing in / up | Screen-local spinner via `isSubmitting` |
| Signed in, first time | Home tab + `DataCollectModal` |
| Signed in, returning | Home tab; `NotificationPermissionsModal` if `pendingPermissions` non-empty |
| Auth error | `AppToast` with mapped Firebase message |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/navigation/index.js` | `AppRouter`: `uuid ? <MainTabNavigator/> : <AuthStack/>` |
| `src/navigation/AuthStack.jsx` | SignUp / SignIn / Forgot native stack |
| `src/screens/SigninScreen`, `SignupScreen`, `ForgotPasswordScreen` | Auth UI |
| `src/actions/userActions.js` | `startLoginWithEmail`, `startSignupWithEmail`, `restoreUUID`, `startLogout`, `startSnedPasswordReserEmail` |
| `src/utils/localStorage.js` | AsyncStorage helpers |
| `src/components/ToSModal` | Terms of Use shown at signup and from Settings |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__integration-test__/__auth-stack-test__/__auth-screen-test__/SignInScreen.test.js` | Sign-in form validation and dispatch |
| `__test__/__integration-test__/__auth-stack-test__/__auth-screen-test__/SignUpScreen.test.js` | Sign-up form validation and dispatch |
| `__test__/__integration-test__/__auth-stack-test__/__auth-navigation-test__/SigninToHome.test.js`, `SignupToHome.test.js` | Auth stack -> main tabs navigation |
| `__test__/__unit-test__/__actions-test__/userActions.unit.test.js` | `startLoginWithEmail`, `startSignupWithEmail`, `startLogout` thunks |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Firebase_Auth]]
- [[Navigation]]
- [[Redux_State]]
- [[Profile_Settings]]
