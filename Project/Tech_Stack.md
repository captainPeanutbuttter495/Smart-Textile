# Tech Stack

Technologies used by the Smart Clothing App on `main`, taken from `SmartClothingApp/package.json` and `functions/package.json`.

---

## Mobile app (`SmartClothingApp/`)

| Technology | Version | Purpose |
|-----------|---------|---------|
| Expo SDK | ~52.0 | Managed native tooling; dev client (`expo-dev-client`), `expo run:android` / `expo run:ios` |
| React Native | 0.76.3 | Cross-platform UI (new architecture **off**: `newArchEnabled=false`) |
| React | 18.3.1 | |
| React Navigation | 6.x (`native`, `bottom-tabs`, `native-stack`) | Auth/main switch, tabs, stacks (`stack` and `material-bottom-tabs` are installed but unused) |
| Redux | 4.2 + `redux-thunk` 2.4 + `react-redux` 8 | Global state; no Redux Toolkit |
| react-native-paper | 5.x | Material Design 3 components; theme in `src/constants/themes.js` |
| Firebase JS SDK | 10.x (`firebase/auth`, `firebase/firestore`) | Auth + Firestore from the client |
| `@react-native-async-storage/async-storage` | 1.23 | Session cache + Firebase Auth persistence |
| `expo-notifications`, `expo-device`, `expo-constants` | 0.28 / 7.0 / 17.0 | Push token registration and handling |
| `react-native-dotenv` | 3.4 (babel plugin, `@env`) | Loads `.env` at build time |
| `react-native-chart-kit` | 6.12 | Line charts on Home + Heart Rate screen |
| `victory-native` + `@shopify/react-native-skia` | 40.0 / 1.5 | Sleep chart, HR recovery scatter, activity rings |
| `react-native-calendars` | 1.13 | Date pickers in `DateToolbar` |
| `react-native-gesture-handler`, `react-native-reanimated` | 2.21 / 3.16 | Swipeable rows in `PermissionsModal` |
| `react-native-vector-icons` | 9.2 | MaterialIcons / FontAwesome5 |
| `@react-native-community/datetimepicker`, `slider`, `@react-native-picker/picker` | | Metrics form controls |
| `react-native-element-dropdown` | 2.10 | `MyDropdown` (two other dropdown libs are installed but unused) |
| `expo-font` + `@expo-google-fonts`-style local TTFs | | Chakra Petch, Poppins, Inter via `useAppFonts` |

### Installed but not used

| Package | Note |
|---------|------|
| `@react-native-firebase/app`, `firestore`, `storage` | Native Firebase; the app uses the JS SDK instead |
| `axios` | `src/network/` scaffold with empty `baseUrl` |
| `d3` | Imported in `ViewSleepData` for `scaleLinear`/`ticks` but the results are unused |
| `firebase-admin`, `firebase-functions` | Belong to `functions/`; also listed in the app package |
| `typescript`, `ts-jest` | No `.ts` files in `src/` |
| `expo-app-loading`, `react-native-splash-screen` | Legacy splash handling |

---

## Cloud Functions (`SmartClothingApp/functions/`)

| Technology | Version | Purpose |
|-----------|---------|---------|
| Node.js | `engines` in `functions/package.json` | Runtime |
| `firebase-functions` | 5.x | Firestore trigger + HTTPS function |
| `firebase-admin` | 12.x | Server-side Firestore access |
| `expo-server-sdk` | 3.10 | Send Expo push messages |
| ESLint (`eslint-config-google`) | 8.x | Predeploy lint |

---

## External services

| Service | Purpose |
|---------|---------|
| Firebase Authentication | Email/password identity |
| Cloud Firestore | System of record, shared with the Coaching Dashboard |
| Firebase Cloud Functions | Invitation push trigger |
| Expo Push Service (EAS project `90fe7b8b-...`) | Push delivery; `EXPO_PROJECT_ID` |
| Expo EAS Build | iOS dev client (`eas.json` profiles `development`, `preview`, `production`) |
| GitHub Actions | `npm test -- -u` on PRs and pushes to `testing` |

Not integrated on `main`: Google Health Connect, Apple HealthKit, Hexoskin API, Sensoria, Bluetooth.

---

## Testing

| Layer | Tools |
|-------|-------|
| App | Jest 29 (`jest-expo` preset), `@testing-library/react-native` 12, `react-test-renderer`, `redux-mock-store`, `flush-promises`; manual mocks in `__test__/__mocks__/` |
| Functions | `firebase-functions-test` is installed; no test files exist |

---

## Related Notes

- [[Project_Structure]]
- [[Testing]]
- [[Environment_Setup]]
- [[SystemDesign/README]]
