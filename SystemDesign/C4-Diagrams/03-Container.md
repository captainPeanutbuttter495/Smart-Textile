# C4 Model - Level 2: Container Diagram

Zooming into the Smart Clothing App system to show its separately runnable / deployable pieces and the Firebase services they depend on.

## Diagram

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    athlete["<b>Athlete</b>"]

    subgraph system["Smart Clothing App"]
        mobile["<b>Mobile App</b><br/>React Native 0.76 + Expo SDK 52<br/>Redux 4 + thunk, React Navigation 6,<br/>react-native-paper (MD3)"]
        storage["<b>AsyncStorage cache</b><br/>uid, firstName, lastName, email,<br/>metricsData, authToken (push token)"]
        functions["<b>Cloud Functions</b><br/>Node.js, firebase-functions v5<br/>functions/index.js"]
    end

    fbauth["<b>Firebase Auth</b><br/>Email/password"]
    firestore["<b>Cloud Firestore</b><br/>Users, Invitations,<br/>Users/uid/* subcollections"]
    expo["<b>Expo Push Service</b>"]
    dashboard["<b>Coaching Dashboard</b><br/>(separate repo)"]

    athlete -->|"Touch UI"| mobile

    mobile -->|"read/write via<br/>src/utils/localStorage.js"| storage
    mobile -->|"firebase/auth<br/>(persisted to AsyncStorage)"| fbauth
    mobile -->|"firebase/firestore"| firestore
    mobile -->|"expo-notifications<br/>getExpoPushTokenAsync"| expo

    firestore -->|"Firestore trigger<br/>Invitations/id onCreate"| functions
    functions -->|"firebase-admin<br/>query Users by email"| firestore
    functions -->|"expo-server-sdk<br/>sendPushNotificationsAsync"| expo
    expo -->|"APNs / FCM"| mobile

    dashboard -->|"creates Invitations,<br/>reads athlete subcollections"| firestore

    style athlete fill:#08427B,color:#fff,stroke:#073B6F
    style mobile fill:#438DD5,color:#fff,stroke:#3579B8
    style storage fill:#438DD5,color:#fff,stroke:#3579B8
    style functions fill:#438DD5,color:#fff,stroke:#3579B8
    style fbauth fill:#999,color:#fff,stroke:#888
    style firestore fill:#999,color:#fff,stroke:#888
    style expo fill:#999,color:#fff,stroke:#888
    style dashboard fill:#999,color:#fff,stroke:#888
    style system fill:#E8E8E8,color:#333,stroke:#BBB
```

### Legend

- **Dark blue** = Users / Actors
- **Light blue** = Containers (individual technical building blocks we build and deploy)
- **Gray** = External systems (Firebase managed services, Expo, the separate dashboard)

## Containers

| Container | Technology | Location | Responsibility |
|-----------|-----------|----------|----------------|
| **Mobile App** | React Native 0.76.3, Expo ~52, React 18.3, Redux 4 + redux-thunk, React Navigation 6, react-native-paper 5, react-native-chart-kit, victory-native + Skia | `SmartClothingApp/App.js`, `src/` | All UI; every Firebase call; push token registration; notification-tap handling |
| **AsyncStorage cache** | `@react-native-async-storage/async-storage` | `src/utils/localStorage.js` | Mirrors identity + metrics so the app can restore the session before Firestore answers; also the Firebase Auth persistence layer |
| **Cloud Functions** | Node.js, `firebase-functions@5`, `firebase-admin@12`, `expo-server-sdk` | `SmartClothingApp/functions/index.js` | `sendInvitationNotification` (Firestore trigger) and a `helloWorld` HTTPS stub |

## Key Relationships

| Relationship | Protocol | Purpose |
|-------------|----------|---------|
| Mobile App -> AsyncStorage | Async key/value | `storeUID` / `getUID`, `storeMetrics` / `getMetrics`, names, email, push token; cleared on logout and account deletion |
| Mobile App -> Firebase Auth | Firebase JS SDK | `initializeAuth(app, { persistence: getReactNativePersistence(AsyncStorage) })` in `firebaseConfig.js` |
| Mobile App -> Firestore | Firebase JS SDK | `doc`, `getDoc`, `setDoc`, `updateDoc`, `deleteDoc`, `arrayUnion`, `arrayRemove`, `query` + `where` + `orderBy` on subcollections |
| Mobile App -> Expo Push | `expo-notifications` | Permission prompt, token retrieval with `EXPO_PROJECT_ID`, Android channel `default` |
| Firestore -> Cloud Functions | Firestore trigger | `functions.firestore.document("Invitations/{invitationId}").onCreate` |
| Cloud Functions -> Expo Push | HTTPS | Push message with `data: {screen: "Home", showPermissionsModal: true, coachId, coachFirstName, coachLastName}` |

## What is not here

- **No application server.** There is no Express/REST layer; `src/network/NetworkManager.js` is an unused axios scaffold with an empty `baseUrl`.
- **No wearable or health-platform container.** Health Connect, HealthKit, Bluetooth, and the Hexoskin API are absent from `package.json` and `src/`. See [[Devices_Tab]] and [[T-001_Android_Health_Connect]].

---

## Related Notes

- [[02-SystemContext]]
- [[04-Component-Frontend]]
- [[05-Component-Backend]]
- [[Firebase_Setup]]
- [[Tech_Stack]]
