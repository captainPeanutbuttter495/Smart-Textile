

What a new team member needs before the app runs: `.env`, native Firebase config files, and the dev-login shortcut.

---

## Prerequisites

- Node.js 18+ (CI uses 18; local machines have run 24)
- Android Studio with an AVD, or a physical Android phone; Xcode + Apple Developer team access for iOS
- `npm install` inside `SmartClothingApp/`
- The `.env` file and `google-services.json` / `GoogleService-Info.plist` from a project admin (all gitignored)

---

## `.env` (loaded by `react-native-dotenv` as `@env`)

| Key | Read by | Note |
|-----|---------|------|
| `FIREBASE_CONFIG_API_KEY` | `firebaseConfig.js` | |
| `FIREBASE_AUTH_DOMAIN` | `firebaseConfig.js` | |
| `FIREBASE_PROJECT_ID` | `firebaseConfig.js` | |
| `FIREBASE_DATABASE_URL` | `firebaseConfig.js` | Realtime Database is unused; value may point at a different project harmlessly |
| `FIREBASE_STORAGE_BUCKET` | `firebaseConfig.js` | Storage is unused (rules deny all) |
| `FIREBASE_MESSANGIND_SENDER_ID` | `firebaseConfig.js` | Spelling is intentional; the code imports this exact name |
| `FIREBASE_APP_ID` | `firebaseConfig.js` | |
| `FIREBASE_MEASUREMENT_ID` | `firebaseConfig.js` | |
| `EXPO_PROJECT_ID` | `src/utils/notifications.js` | Must equal `app.json` `extra.eas.projectId` (`90fe7b8b-b25e-4389-8373-0f4ba76b7b02`) or push tokens fail |

Any Google OAuth client ids or `EXPO_PUBLIC_*` keys in an admin handout belong to other projects; Google sign-in was never wired up here.

After editing `.env`, restart Metro with `npx expo start --clear` (dotenv values are inlined at build time).

---

## Native config files

| File | Location | Used by |
|------|----------|---------|
| `google-services.json` | `SmartClothingApp/android/app/` | Android dev build (`app.json` `android.googleServicesFile`) |
| `GoogleService-Info.plist` | `SmartClothingApp/` | iOS build (`app.json` `ios.googleServicesFile`) |

Without `google-services.json`, `npm run android` fails at the Gradle step; Expo Go still works for everything except push notifications.

---

## Running

| Goal | Command |
|------|---------|
| Metro for Expo Go (no push) | `npx expo start` (see [[Android_Emulator_Workflow]] for the `--offline --localhost` flags this laptop needs) |
| Android dev build (push works) | `npm run android` |
| iOS dev build | `eas build --platform ios --profile development` then `npx expo start` (see [[iOS_EAS_Build]]) |
| Clear stale cache ("Main not registered") | `npx expo start --clear` |
| Tests | `npm test` |

---

## Dev login bypass

`src/navigation/index.js` shows the main app whenever `state.user.uuid` is non-null. To skip the login screen while developing UI, set `uuid` to any string in `initialState` of `src/reducers/userReducer.js`. Firestore calls will still fail without a real session, so this is only useful for layout work.

---

## Related Notes

- [[Firebase_Setup]]
- [[Android_Emulator_Workflow]]
- [[iOS_EAS_Build]]
- [[Project_Structure]]
- [[Push_Notifications_Client]]
