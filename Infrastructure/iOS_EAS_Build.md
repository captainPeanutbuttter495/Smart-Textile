# iOS EAS Build

Building and running the iOS development client with EAS. Required for push notifications on iOS; Expo Go does not work for this project on iOS.

---

## Overview

From the repository README: iOS testing needs an Apple Developer account, an EAS **development** build, and a Metro server. Team members are added to the Apple Developer Team by the project owner (App Store Connect -> Users and Access, Developer role) instead of sharing credentials. Ask the admin listed in the README for service-account and `.env` details related to push tokens.

---

## Steps

```bash
cd SmartClothingApp
npm install
npm install -g eas-cli          # if not present
eas login
eas build --platform ios --profile development   # Terminal 1: builds the dev client in the cloud
npx expo start --clear                           # Terminal 2: Metro
```

Install the resulting build on a physical device (simulator builds are disabled: `eas.json` sets `ios.simulator: false` for the development profile), open it, and connect to Metro.

---

## `eas.json` profiles

| Profile | Settings | Use |
|---------|----------|-----|
| `development` | `developmentClient: true`, `distribution: internal`, `ios.simulator: false` | Day-to-day device testing with push |
| `preview` | `distribution: internal` | Shareable internal build |
| `production` | `autoIncrement: true` | Store build (never used to date) |

`cli.appVersionSource` is `remote`; `app.json` `runtimeVersion` is `1.0.0` and `updates.url` points at the EAS project.

---

## `app.json` iOS settings

| Key | Value |
|-----|-------|
| `bundleIdentifier` | `com.arcs.smartclothingapp` |
| `googleServicesFile` | `./GoogleService-Info.plist` (gitignored) |
| `entitlements.aps-environment` | `development` |
| `infoPlist.ITSAppUsesNonExemptEncryption` | `false` |
| `infoPlist.CFBundleDisplayName` | `SmartClothing` |

`ios/` is committed; regenerate with `npx expo prebuild --platform ios` when native config changes.

---

## Not on iOS today

Apple HealthKit is not integrated on `main`: no HealthKit entitlement (only `aps-environment`), no `NSHealth*UsageDescription`, nothing under `src/`.

Two abandoned branches hold a March-2024 prototype: `origin/ios-dev` has a hand-written Swift `Controller` native module (`ios/Controller.swift` + `Controller.m`, legacy bridge) and a Firestore upload service; `origin/ios-dev-carlos` is the same code plus two local build fix-up commits (Podfile.lock, dev team ID, one `Info.plist` string). Both are on Expo 50 / RN 0.73 and ~690 commits behind `main`, and the data path had field-name and duplicate-write bugs that meant it never worked end to end. Reference only, never merge; the full assessment and the fresh-build plan are in [[T-002_iOS_Apple_HealthKit]].

---

## Related Notes

- [[Environment_Setup]]
- [[Android_Emulator_Workflow]]
- [[Push_Notifications_Client]]
- [[T-002_iOS_Apple_HealthKit]]
