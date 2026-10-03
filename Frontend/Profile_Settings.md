# Profile & Settings

Profile screen, the body-metrics form, and the Settings screen with its account modals.

---

## Overview

Both screens are reached from the `AppHeader` overflow menu ("Edit Profile", "Settings & Privacy"). Profile shows identity + metrics; Settings hosts account-level actions as modals.

---

## Profile (`ProfileScreen`)

| Element | Source | Edit path |
|---------|--------|-----------|
| Name | `state.user.firstName` / `lastName` | `PersonalModal` -> `startUpdateProfile(first, last)` (updates Auth `displayName` + AsyncStorage + Redux) |
| Age | `userMetricsData.dob` (Firestore `Timestamp` converted with `toDate()`), computed client-side | `DataCollectModal` |
| Gender, Sport, Height (`ft'in"`), Weight (lbs) | `state.user.userMetricsData` | `DataCollectModal` via `userMetricsDataModalVisible(true, false)` |

`state.app.measurementSystem` (`"imperial"` default) exists in the reducer but no UI toggles it and no screen reads it; values are always shown imperial with metric in parentheses inside the modal.

---

## `DataCollectModal`

`src/components/DataCollectModal/index.jsx`, mounted on `HomeScreen`, visible when `state.app.userMetricsDataModalVisible`.

| Field | Control | Values |
|-------|---------|--------|
| Gender | `MyDropdown` | `Gender` map in `src/utils/metrics.js` |
| Sports | `MyDropdown` | `Sports` map in `src/utils/metrics.js` |
| Date of Birth | `DateTimePickerAndroid.open` (Android API only; iOS has no picker here) | `Date` |
| Height | `Slider` 0-118 in | shown as `5'11" (180 cm)` |
| Weight | `Slider` 0-800 lb | shown as `165 lbs (75 kg)` |

- **Save** -> `startUpdateUserData({gender, dob, height, weight, sports})` -> `updateDoc(Users/{uid})`, `updateUserMetricsData`, `storeMetrics`.
- **Skip / Cancel** -> resets local state to empty values and *also* calls `handleSubmit`, so an empty-ish record is written (comment in code: "otherwise user uid doc doesn't show").
- On open (not from sign-up) it pre-fills from `userMetricsData` when `gender !== "No Data"`.

---

## Settings (`SettingsScreen`)

| Button | Modal / action | Firebase calls |
|--------|---------------|----------------|
| UPDATE EMAIL | `UpdateEmailModal` | `reauthenticate(password)` then `updateUserEmail(newEmail)`; Firestore `Users/{uid}.email` is not updated |
| CHANGE PASSWORD | `ChangePasswordModal` | `reauthenticate` then `updateUserPassword` |
| MANAGE PERMISSIONS | `PermissionsModal` | See [[Coach_Access_Permissions]] |
| DELETE DATA | `SettingsButton` with no `onPress` | **Not implemented** |
| DELETE ACCOUNT | `DeleteAccountModal` | `reauthenticate` then `deleteAccount()` (Auth user + `Users/{uid}` doc; subcollections remain) |
| TERMS OF SERVICES | `ToSModal` | Static text (mentions Hexoskin, Sensoria, Apple/Android watches) |

Logout lives in the `AppHeader` menu, not on this screen.

---

## Accessibility (`AccessibilityScreen`)

Placeholder: renders the word "Accessibility" and nothing else.

---

## Screen States

| State | Display |
|-------|---------|
| Metrics never entered | Profile shows `No Data` for gender / height / weight / sport |
| Modal submit | Firestore write; toast only on the email flow |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/screens/Profile/index.jsx` | Profile view, age + height formatting |
| `src/components/PersonalModal/PersonalModal.jsx` | First / last name editor |
| `src/components/DataCollectModal/index.jsx` | Metrics form |
| `src/utils/metrics.js` | `Gender`, `Sports`, `Height` maps |
| `src/screens/Settings/index.jsx` | Settings buttons + modal visibility state |
| `src/components/UpdateEmailModal`, `ChangePasswordModal`, `DeleteAccountModal`, `ToSModal` | Account modals |
| `src/components/UI/SettingsButton.jsx` | Row button |
| `src/screens/Accessibility/index.jsx` | Placeholder |

---

## Tests

| Suite | Covers |
|-------|--------|
| `__test__/__integration-test__/__main-tab-test__/__main-screen-test__/SettingsScreen.test.js` | Settings render, delete account, snapshot |
| `__test__/__integration-test__/__main-tab-test__/__main-navigation-test/HomeToProfile.test.js` | Home -> Profile -> Edit Personal Info / Metrics Data navigation |
| `__test__/__unit-test__/__components-test__/AppHeader.unit.test.js` | Header render + logout |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Authentication]]
- [[Firebase_Auth]]
- [[Coach_Access_Permissions]]
- [[Redux_State]]
