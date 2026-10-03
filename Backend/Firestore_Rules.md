# Firestore Rules

The security rules deployed with this repository, what they allow, and the gap they leave.

---

## Overview

`SmartClothingApp/firestore.rules` (deployed via `firebase.json` -> `firestore.rules`):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /Users/{userId} {
      allow read, write: if request.auth != null;
    }
    match /Invitations/{invitationId} {
      allow read, write: if request.auth != null;
    }
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

Every rule is the same: **any signed-in user can read and write any document**, including other athletes' profiles, push tokens, coach lists, and health subcollections. The catch-all `match /{document=**}` makes the first two blocks redundant.

`storage.rules` denies all access (`allow read, write: if false`); the app does not use Cloud Storage.

---

## Why it is this way

The Coaching Dashboard (a separate repo, same Firebase project) needs coaches to read athletes' documents and to append to `Users/{athleteUid}.pendingPermissions`. Rather than model that, the rules were opened for all authenticated users. The dashboard's Insights API uses `firebase-admin`, which bypasses rules anyway.

---

## Gap to ticket

| Risk | Detail |
|------|--------|
| Cross-athlete reads | Any athlete account can `getDoc(Users/{otherUid})` and read health subcollections |
| Cross-athlete writes | Any athlete account can overwrite another's `coachList`, `expoPushToken`, or metrics |
| Invitations | Any athlete can create invitations, triggering push notifications to any email |

A tightened rule set would need: `Users/{uid}` read/write only when `request.auth.uid == uid`; coaches (`role == "coach"`) allowed to read athlete docs they appear in `coachList` of and to `arrayUnion` themselves into `pendingPermissions`; `Invitations` create only for `role == "coach"`. This has to be coordinated with the dashboard team since their SPA writes with the Web SDK (rules apply).

---

## Deploying

```bash
cd SmartClothingApp
firebase deploy --only firestore:rules
```

`firestore.indexes.json` is empty; the app's queries are single-field range filters (`date`, `startDate`) that need no composite index.

---

## Key Files

| File | Purpose |
|------|---------|
| `firestore.rules` | Firestore access rules |
| `storage.rules` | Cloud Storage rules (deny all) |
| `firestore.indexes.json` | Composite indexes (none) |
| `firebase.json` | Points the CLI at the three files above and the `functions/` codebase |

---

## Tests

| Suite | Covers |
|-------|--------|
| none | No rules tests; the Firebase emulator is not set up |

See [[Testing]] for tooling and mocks.

---

## Related Notes

- [[Firestore_Schema]]
- [[Cloud_Functions]]
- [[Firebase_Setup]]
- [[Coach_Access_Permissions]]
