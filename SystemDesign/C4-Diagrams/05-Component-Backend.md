# C4 Model - Level 3b: Component Diagram (Backend)

Inside the Firebase side of the system: the Firestore collections the app depends on, the security rules that guard them, and the one Cloud Function deployed from this repository. The Coaching Dashboard is shown because it reads and writes the same documents.

## Diagram

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    mobile["<b>Mobile App</b><br/>userActions.js"]
    dashboard["<b>Coaching Dashboard</b><br/>SPA + Insights API<br/>(separate repo)"]

    subgraph firestore["Cloud Firestore"]
        rules["<b>firestore.rules</b><br/>allow read, write:<br/>if request.auth != null<br/>(every collection)"]
        users["<b>Users/uid</b><br/>profile, metrics, expoPushToken,<br/>pendingPermissions, coachList,<br/>disabledCoachList, role"]
        hr["<b>Users/uid/HeartRateData</b><br/>date, heartRate<br/>read by app"]
        sleep["<b>Users/uid/SleepData</b><br/>startDate, endDate, sleepValue<br/>read by app"]
        hrhc["<b>Users/uid/HeartRateDataHC</b><br/>time, beatsPerMinute<br/>read by dashboard"]
        sleephc["<b>Users/uid/SleepDataHC</b><br/>startTime, endTime, stage<br/>read by dashboard"]
        vo2["<b>Users/uid/VO2MaxDataHX</b><br/>read by dashboard"]
        invites["<b>Invitations/id</b><br/>athleteEmail, coachId,<br/>coachFirstName, coachLastName"]
    end

    subgraph functions["Cloud Functions (functions/index.js)"]
        trigger["<b>sendInvitationNotification</b><br/>firestore.document('Invitations/invitationId').onCreate"]
        lookup["<b>Users lookup</b><br/>where('email', '==', athleteEmail)"]
        push["<b>sendPushNotification()</b><br/>expo.sendPushNotificationsAsync"]
        hello["<b>helloWorld</b><br/>https.onRequest stub"]
    end

    expo["<b>Expo Push Service</b>"]

    mobile -->|"setDoc / updateDoc / getDoc"| users
    mobile -->|"query where date between"| hr
    mobile -->|"query where startDate between"| sleep
    dashboard -->|"addDoc"| invites
    dashboard -->|"set role"| users
    dashboard -->|"firebase-admin reads"| hrhc
    dashboard -->|"firebase-admin reads"| sleephc
    dashboard -->|"firebase-admin reads"| vo2

    invites -->|"onCreate"| trigger
    trigger --> lookup
    lookup -->|"admin.firestore()"| users
    lookup -->|"expoPushToken"| push
    push --> expo

    rules -.->|"guards"| users
    rules -.->|"guards"| invites

    style mobile fill:#1168BD,color:#fff,stroke:#0E5CA4
    style dashboard fill:#999,color:#fff,stroke:#888
    style rules fill:#438DD5,color:#fff,stroke:#3579B8
    style users fill:#438DD5,color:#fff,stroke:#3579B8
    style hr fill:#438DD5,color:#fff,stroke:#3579B8
    style sleep fill:#438DD5,color:#fff,stroke:#3579B8
    style hrhc fill:#999,color:#fff,stroke:#888
    style sleephc fill:#999,color:#fff,stroke:#888
    style vo2 fill:#999,color:#fff,stroke:#888
    style invites fill:#438DD5,color:#fff,stroke:#3579B8
    style trigger fill:#438DD5,color:#fff,stroke:#3579B8
    style lookup fill:#438DD5,color:#fff,stroke:#3579B8
    style push fill:#438DD5,color:#fff,stroke:#3579B8
    style hello fill:#438DD5,color:#fff,stroke:#3579B8
    style expo fill:#999,color:#fff,stroke:#888
    style firestore fill:#E8E8E8,color:#333,stroke:#BBB
    style functions fill:#E8E8E8,color:#333,stroke:#BBB
```

### Legend

- **Blue** = The mobile app (caller)
- **Light blue** = Components this repository defines or depends on directly (collections it reads/writes, rules, functions)
- **Gray** = External systems, and collections that only the Coaching Dashboard uses today

## Components

| Component | Defined in | Responsibility |
|-----------|-----------|----------------|
| **`Users/{uid}`** | Created by `startSignupWithEmail`; updated by profile / metrics / coach-access thunks | Single document per athlete. Also carries `role` written by the dashboard's admin tools |
| **`Users/{uid}/HeartRateData`, `SleepData`** | Read by `queryHeartRateData`, `querySleepData` | Health time-series in the shape the app's Health tab parses. **No writer exists in this repo.** |
| **`Users/{uid}/HeartRateDataHC`, `SleepDataHC`, `VO2MaxDataHX`** | Read by `coaching-dashboard/server/server.js` | Health time-series in raw Health Connect / Hexoskin shapes. **Not read or written by this app.** |
| **`Invitations/{id}`** | Written by the Coaching Dashboard | Triggers the push; the app never reads it |
| **`firestore.rules`** | `SmartClothingApp/firestore.rules` | Any authenticated user may read and write any document (catch-all `match /{document=**}`) |
| **`sendInvitationNotification`** | `functions/index.js` | On invite create: find the athlete by `email`, and if they have an `expoPushToken`, send "Allow Coach {name} to Track Your Data?" with the navigation payload |
| **`helloWorld`** | `functions/index.js` | Default HTTPS stub, unused |

## Key Relationships

| Relationship | Protocol | Purpose |
|-------------|----------|---------|
| Mobile App -> `Users/{uid}` | Firebase JS SDK | `setDoc` at signup; `updateDoc` for metrics, `expoPushToken`, `arrayUnion` / `arrayRemove` on `pendingPermissions` + `coachList`; `deleteDoc` on account deletion |
| Mobile App -> health subcollections | Firebase JS SDK `query` | `where("date", ">=", start)`, `where("date", "<=", end)`; sleep adds `orderBy("startDate")` and a client-side `endDate` filter |
| Dashboard -> Firestore | Firebase Web SDK + firebase-admin | Invitations, role changes, per-athlete time-series reads |
| Invitations -> Cloud Function -> Expo | Firestore trigger, HTTPS | Real-time coach invite notification |

## Known gaps

- **Schema divergence.** The app and the dashboard read different subcollections with different field names. Any ingestion work must pick one contract; see [[Firestore_Schema]] and [[T-001_Android_Health_Connect]].
- **Rules are wide open** for authenticated users; an athlete can read any other athlete's document. See [[Firestore_Rules]].
- `disabledCoachList` is written at signup but `fetchCoachAccess` reads `CoachListDisabled` (different casing); the disable feature is partially wired.

---

## Related Notes

- [[03-Container]]
- [[06-Code]]
- [[Firestore_Schema]]
- [[Firestore_Rules]]
- [[Cloud_Functions]]
