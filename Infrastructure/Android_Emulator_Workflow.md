
Known-good way to run the app on the Android emulator on the team's Windows laptops, and the failure modes to recognise.

---

## Expo Go (no push notifications)

```bash
cd SmartClothingApp
npx expo start --offline --localhost
# press  s  if the CLI says "Using development build", then  a
```

| Flag | Why |
|------|-----|
| `--offline` | `app.json` carries an EAS `projectId`, so the CLI wants an Expo login to sign the manifest; offline mode skips it |
| `--localhost` | The emulator's NAT route (`exp://<LAN-IP>:8081`) silently corrupts the ~17 MB dev bundle. Symptoms: okhttp `ProtocolException: Expected leading [0-9a-fA-F] character`, Expo Go hangs ~120 s then shows "Sorry about that". Over `adb reverse` (which `--localhost` uses) the download is byte-perfect |

Diagnose a hang with:

```bash
adb logcat --pid=$(adb shell pidof host.exp.exponent) | grep -E "ProtocolException|UpdateFailedToLoad"
adb shell am force-stop host.exp.exponent   # the error screen is modal; force-stop then relaunch
```

Give the AVD at least 4 GB RAM (`hw.ramSize=4096` in the AVD `config.ini`); at 2 GB the guest swaps and Metro reloads time out.

---

## Dev build (push notifications work)

```bash
cd SmartClothingApp
npm run android          # expo run:android -> Gradle build -> installs com.arcs.smartclothingapp
npx expo start           # Metro for the dev client
```

Requires `android/app/google-services.json` from an admin. `plugins/customNamespacePlugin.js` runs during `expo prebuild` to remove the `package` attribute from `AndroidManifest.xml` and force the Gradle `namespace` (Expo 52 / AGP 8 requirement).

---

## Common errors

| Error | Fix |
|-------|-----|
| "Main has not been registered" | `npx expo start --clear` |
| Gradle: missing `google-services.json` | Get the file; it is gitignored |
| `EXPO_PROJECT_ID` undefined at runtime | `.env` missing or Metro started before the file existed; restart with `--clear` |
| Health Connect / native module errors | Not applicable on `main`; will apply once [[T-001_Android_Health_Connect]] adds `react-native-health-connect` (requires a dev build, not Expo Go) |

---

## Related Notes

- [[Environment_Setup]]
- [[iOS_EAS_Build]]
- [[Push_Notifications_Client]]
