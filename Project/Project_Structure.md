# Project Structure

Repository layout for the Smart Clothing App. The git root is `smart-clothing-app/`; the Expo project lives one level down in `SmartClothingApp/`, and all `npm` commands run from there.

---

## Repository

```
smart-clothing-app/                 git root
├── README.md                       project README (goals, push setup, iOS instructions)
├── LICENSE
├── package-lock.json               stray root lockfile (unused)
├── .github/workflows/main.yml      CI: npm test -- -u
└── SmartClothingApp/               the Expo / React Native app
```

---

## `SmartClothingApp/`

```
SmartClothingApp/
├── App.js                          boot: store, fonts, auth restore, push, notification listeners
├── index.js                        registerRootComponent
├── firebaseConfig.js               initializeApp / initializeAuth / getFirestore
├── app.json                        Expo config: scheme, icons, plugins, EAS projectId
├── eas.json                        EAS build profiles
├── babel.config.js                 babel-preset-expo + react-native-dotenv (@env) + reanimated
├── package.json                    scripts: start, android, ios, web, test
├── .env                            Firebase + EXPO_PROJECT_ID (gitignored, ask an admin)
├── firebase.json                   functions codebase, rules + indexes paths
├── firestore.rules / storage.rules / firestore.indexes.json
├── CLAUDE.md                       AI assistant guidance for this repo
├── .claude/skills/docs-sync/       vault + diagram conventions
├── plugins/customNamespacePlugin.js  Expo config plugin: strip manifest package attr, force Gradle namespace
├── android/, ios/                  committed native projects (prebuild output)
├── assets/                         icons, splash, fonts/, images/ (sock.jpeg, shirt.jpeg)
├── functions/                      Firebase Cloud Functions package (own package.json)
│   └── index.js                    sendInvitationNotification, helloWorld
├── docs/                           generated JSDoc output (gitignored, unrelated to the vault)
├── __test__/                       Jest tests (see Testing)
└── src/
    ├── actions/                    userActions, appActions, deviceActions, toastActions, types
    ├── reducers/                   userReducer, appReducer, deviceReducer, toastReducer
    ├── store.js                    combineReducers + thunk
    ├── navigation/                 index (AppRouter), AuthStack, HomeStack (tabs + AllNavigationStack)
    ├── screens/                    one folder per screen + index.js barrel
    ├── components/                 shared UI + index.js barrel
    │   ├── AppHeader/              header + overflow menu
    │   ├── AppCards/               ActivityCard, DeviceCard
    │   ├── DateToolbar/            BaseCalendar, DatePicker, DateRangePicker, DateToolbar
    │   ├── Dialogs/                AppToast, PromptModal
    │   ├── UI/                     DataButton, LoadingOverlay, SettingsButton, dropdown
    │   ├── visualizations/         ActivityChart, ActivityRings, BreathingRateChart, HeartRateChart, VentilationChart
    │   ├── DailyInsights/, DailyMetrics/
    │   └── *Modal/                 DataCollect, Permissions, NotificationPermissions, Personal, ToS, UpdateEmail, ChangePassword, DeleteAccount
    ├── constants/                  themes.js (MD3 theme, colors, fonts, styles), errors.js
    ├── hooks/                      useAppFonts
    ├── utils/                      localStorage, notifications, calendar, dateConversions, metrics, scale, firebaseErrorMessages
    ├── network/                    unused axios NetworkManager + config
    └── services/                   empty (export {})
```

---

## Conventions

- **Screens** live in `src/screens/<Name>/index.jsx` and are re-exported from `src/screens/index.js`; import them as `import { HomeScreen } from "../screens"`.
- **Components** follow the same barrel pattern via `src/components/index.js`.
- **Actions**: plain creators `fooBar`, thunks `startFooBar`; all Firebase calls in `src/actions/userActions.js`.
- **Styling**: `react-native-paper` components + `StyleSheet.create`; shared tokens from `src/constants/themes.js` (`AppColor`, `AppFonts`, `AppStyle`, `AppTheme`).
- **Native config**: `app.json` is the source of truth; `android/` and `ios/` are committed and regenerated with `npx expo prebuild` (the custom plugin fixes the Gradle namespace).
- **Env**: values come from `.env` via `import { X } from "@env"`.

---

## Related Notes

- [[Tech_Stack]]
- [[Testing]]
- [[Environment_Setup]]
- [[Navigation]]
- [[Redux_State]]
