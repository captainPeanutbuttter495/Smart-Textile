# C4 Model - Level 0: System Landscape Diagram

The broadest view: every person and software system in the ARCS smart-textile research ecosystem, and how the Smart Clothing App fits alongside the separately-developed Coaching Dashboard.

## Diagram

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    athlete["<b>Athlete</b><br/>CSUN student-athlete<br/>Signs up, enters body metrics,<br/>manages coach access"]
    coach["<b>Coach</b><br/>Invites athletes, reviews<br/>team + athlete trends"]
    admin["<b>Admin</b><br/>Manages user roles<br/>(dashboard only)"]

    app["<b>Smart Clothing App</b><br/>React Native + Expo mobile app<br/>(this repository)"]

    dashboard["<b>Coaching Dashboard</b><br/>React SPA + Insights API + email service<br/>(separate repository)"]

    subgraph firebase["Firebase project (shared)"]
        fbauth["<b>Firebase Auth</b><br/>Email/password identity"]
        firestore["<b>Cloud Firestore</b><br/>Users, Invitations,<br/>per-athlete subcollections"]
        functions["<b>Cloud Functions</b><br/>sendInvitationNotification"]
    end

    expo["<b>Expo Push Service</b><br/>Delivers push notifications<br/>to athlete devices"]

    wearables["<b>Wearables / Smart garments</b><br/>Health Connect, HealthKit, Hexoskin<br/><i>(NOT IMPLEMENTED on main)</i>"]

    athlete -->|"Uses"| app
    coach -->|"Uses"| dashboard
    admin -->|"Uses"| dashboard
    athlete -.->|"Accepts invite (web)"| dashboard

    app -->|"Firebase JS SDK"| fbauth
    app -->|"Firebase JS SDK"| firestore
    app -->|"Registers Expo push token"| expo

    dashboard -->|"Firebase Web SDK + firebase-admin"| fbauth
    dashboard -->|"Writes Invitations,<br/>reads athlete data"| firestore

    firestore -->|"onCreate Invitations/id"| functions
    functions -->|"expo-server-sdk"| expo
    expo -->|"Push notification"| app

    wearables -.->|"No data path exists yet"| app

    style athlete fill:#08427B,color:#fff,stroke:#073B6F
    style coach fill:#08427B,color:#fff,stroke:#073B6F
    style admin fill:#08427B,color:#fff,stroke:#073B6F
    style app fill:#1168BD,color:#fff,stroke:#0E5CA4
    style dashboard fill:#999,color:#fff,stroke:#888
    style fbauth fill:#999,color:#fff,stroke:#888
    style firestore fill:#999,color:#fff,stroke:#888
    style functions fill:#438DD5,color:#fff,stroke:#3579B8
    style expo fill:#999,color:#fff,stroke:#888
    style wearables fill:#999,color:#fff,stroke:#888,stroke-dasharray: 5 5
    style firebase fill:#E8E8E8,color:#333,stroke:#BBB
```

### Legend

- **Dark blue** = Users / Actors
- **Blue** = Smart Clothing App system (what we build)
- **Light blue** = Container we own inside a shared service (the Cloud Function lives in this repo's `functions/`)
- **Gray** = External systems (third-party, or the separate Coaching Dashboard repo)
- **Gray dashed node + dashed arrow** = *(planned)*, not implemented on `main`; shown only so the gap is visible
- **Light border** = boundary subgraph

## Key Relationships

| Relationship | Protocol | Purpose |
|-------------|----------|---------|
| Athlete -> Smart Clothing App | Touch UI | Sign up / log in, enter metrics, view health screens, manage coach permissions |
| Coach / Admin -> Coaching Dashboard | HTTPS | Invite athletes, view aggregated data, manage roles |
| Smart Clothing App -> Firebase Auth | Firebase JS SDK | `createUserWithEmailAndPassword`, `signInWithEmailAndPassword`, `onAuthStateChanged`, reauth |
| Smart Clothing App -> Firestore | Firebase JS SDK | `Users/{uid}` profile + metrics, `pendingPermissions` / `coachList`, `expoPushToken`; reads `HeartRateData` / `SleepData` |
| Coaching Dashboard -> Firestore | Firebase Web SDK (SPA), firebase-admin (API) | Writes `Invitations/{id}`, sets `Users/{uid}.role`, reads `HeartRateDataHC` / `SleepDataHC` / `VO2MaxDataHX` |
| Firestore -> Cloud Functions | Firestore trigger | `Invitations/{id}` `onCreate` fires `sendInvitationNotification` |
| Cloud Functions -> Expo Push | HTTPS (`expo-server-sdk`) | Sends the "Allow Coach ... to Track Your Data?" notification |
| Expo Push -> Smart Clothing App | APNs / FCM | Delivers notification; tap opens `NotificationPermissionsModal` |

## Systems

| System | Repository | Status |
|--------|-----------|--------|
| **Smart Clothing App** | `smart-clothing-app/SmartClothingApp` | Active — this vault documents it |
| **Coaching Dashboard** | `coaching-dashboard` (sibling repo) | Active, separate team; reads the same Firestore project |
| **Cloud Functions** | `smart-clothing-app/SmartClothingApp/functions` | Deployed from this repo |
| **Wearable ingestion** | — | Not implemented; see [[T-001_Android_Health_Connect]] |

---

## Related Notes

- [[00-C4-Overview]]
- [[02-SystemContext]]
- [[Roadmap]]
- [[T-001_Android_Health_Connect]]
- [[T-002_iOS_Apple_HealthKit]]
