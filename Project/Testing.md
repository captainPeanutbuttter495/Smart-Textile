# Testing

Test tooling, folder conventions, mocking patterns, the current suite inventory, and how CI runs them.

---

## Overview

Jest 29 with the `jest-expo` preset (config inline in `package.json`). Run from `SmartClothingApp/`:

```bash
npm test                                   # all suites
npm test -- -u                             # update snapshots (what CI runs)
npx jest __test__/__unit-test__/__reducers-test__/userReducer.unit.test.js
npx jest -t "pattern"                      # by test name
```

There is no lint script for the app; `eslint.config.js` exists but is not wired to any script or CI step.

---

## Folder layout

```
__test__/
├── __mocks__/                       manual module mocks (auto-picked by jest.mock('<name>'))
│   ├── expo-notifications.js
│   ├── expo-font.js
│   ├── @shopify__react-native-skia.js
│   ├── react-native-vector-icons.js
│   └── AppFontsResource.js
├── __unit-test__/
│   ├── __actions-test__/            appActions, userActions
│   ├── __reducers-test__/           userReducer, toastReducer
│   ├── __components-test__/         AppHeader (+ snapshot)
│   └── __utils-test__/              localStorage, notifications
├── __integration-test__/
│   ├── __auth-stack-test__/         SignIn/SignUp screens, hooks, Signin->Home, Signup->Home
│   ├── __main-tab-test__/           HomeToProfile, HomeToSettings, SettingsScreen, ViewInsightsScreen, ViewSleepData (+ snapshots)
│   └── userActions.integration.test.js
├── archives/                        retired tests (SigninScreen_test, SignupScreen_test)
├── App_test.js                      fully commented out
└── Profile_test.js                  fully commented out
```

---

## Patterns

| Pattern | Where | How |
|---------|-------|-----|
| Mock Firebase at the module boundary | Integration tests | `jest.mock('../../../../firebaseConfig.js', () => ({ auth: {...}, database: {...} }))` plus `jest.mock('firebase/firestore', ...)` and `jest.mock('firebase/auth', ...)` factories at the top of the file. Never hit the real SDK |
| Mock AsyncStorage helpers | Integration tests | `jest.mock('../../../../src/utils/localStorage.js', () => ({ getUID: jest.fn(), ... }))` |
| Fresh module graph per test | `notifications.unit.test.js` | `jest.resetModules()` in `beforeEach`, then `require()` the SUT and its mocked deps inside the test |
| Manual mocks | `__test__/__mocks__/` | `jest.mock('expo-notifications')` with no factory picks up the file there |
| Real store + navigation | Screen integration tests | Render inside `<Provider store={configureStore()}>` and `<NavigationContainer>`; assert with `@testing-library/react-native` |
| Thunk tests | `userActions.unit.test.js` | `redux-mock-store` + `flush-promises` |
| Snapshots | Committed under `__snapshots__/` | CI runs with `-u`, so snapshot drift never fails a build |

Known limitation: three `projectId` cases in `notifications.unit.test.js` are commented out (the `@env` import could not be mocked reliably).

---

## Suite inventory (main)

| Suite | Type | Covers |
|-------|------|--------|
| `appActions.unit.test.js` | unit | Modal flags, date-range creators + thunks |
| `userActions.unit.test.js` | unit | Auth/profile thunks with mocked Firebase, `querySleepData` |
| `userReducer.unit.test.js`, `toastReducer.unit.test.js` | unit | Reducer cases |
| `AppHeader.unit.test.js` | unit + snapshot | Header render |
| `localStorage.unit.test.js` | unit | AsyncStorage helpers |
| `notifications.unit.test.js` | unit | Permission + token flow |
| `SignInScreen.test.js`, `SignUpScreen.test.js`, `SignInHooks-test.js` | integration | Auth screens |
| `SigninToHome.test.js`, `SignupToHome.test.js` | integration | Auth -> Main navigation |
| `HomeToProfile.test.js`, `HomeToSettings.test.js` | integration | Header menu navigation |
| `SettingsScreen.test.js`, `ViewInsightsScreen.test.js`, `ViewSleepData.test.js` | integration + snapshot | Screen renders with mocked Firebase |
| `userActions.integration.test.js` | integration | Thunks against mocked Firestore |

No tests exist for `functions/` (`firebase-functions-test` is installed but unused).

---

## CI

`.github/workflows/main.yml` runs on pull requests, pushes to `testing`, and manual dispatch: Node 18, `npm install`, `npm test -- -u` in `SmartClothingApp/`. See [[CI_GitHub_Actions]].

---

## Adding tests for new features

- Screen test: copy the `jest.mock` block from `ViewSleepData.test.js`, render with a real store.
- Native module (e.g. `react-native-health-connect`): add `__test__/__mocks__/<package>.js` so Jest never loads the native binding.
- Thunk: mock `firebaseConfig.js` and `firebase/firestore`, assert dispatched actions with `redux-mock-store`.

---

## Related Notes

- [[Tech_Stack]]
- [[Project_Structure]]
- [[CI_GitHub_Actions]]
- [[Redux_State]]
