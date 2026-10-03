# C4 Model - Dynamic Diagrams

Runtime behavior of the key interaction flows on `main`: login and session restore, sign-up, the coach invite push, the health-tab query, and logout.

Every message sits inside a dark `rect` block, because the shared init line renders message text white. Colours mean the same thing in every diagram below.

---

## 1. Login + Session Restore

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    actor Athlete
    participant App as App.js
    participant Store as Redux store
    participant Screen as SigninScreen
    participant Actions as userActions.js
    participant Auth as Firebase Auth
    participant FS as Firestore
    participant AS as AsyncStorage

    rect rgb(46, 110, 70)
    Note over Athlete,AS: First login
    Athlete->>Screen: Enter email + password, tap Sign In
    Screen->>Actions: dispatch startLoginWithEmail(email, password)
    Actions->>Auth: signInWithEmailAndPassword
    Auth-->>Actions: userCredential
    Actions->>AS: storeUID, storeFirstName, storeLastName, storeEmail
    Actions->>Actions: dispatch startLoadUserData()
    Actions->>FS: getDoc Users/uid
    FS-->>Actions: profile + metrics
    Actions->>Store: updateUserMetricsData(data)
    Actions->>Store: loginWithEmail(uuid, firstName, lastName, email)
    Store-->>App: state.user.uuid set
    App->>Athlete: AppRouter renders MainTabNavigator
    end

    rect rgb(40, 85, 140)
    Note over Athlete,AS: Cold start with an existing session
    App->>Auth: onAuthStateChanged(auth, cb)
    Auth-->>App: user, persisted via getReactNativePersistence
    App->>AS: getUID()
    AS-->>App: uid
    App->>Store: dispatch restoreUUID()
    App->>AS: getMetrics()
    alt metrics cached
        App->>Store: updateUserMetricsData(cached)
    else no cache
        App->>Actions: dispatch startLoadUserData()
        Actions->>FS: getDoc Users/uid
    end
    App->>App: checkPendingPermissions(user)
    App->>Actions: dispatch startRegisterPushToken()
    App->>Athlete: MainTabNavigator
    end
```

### Legend

- **Green** = first login through `SigninScreen`
- **Blue** = cold start, where the AsyncStorage `uid` decides whether the athlete is logged in

---

## 2. Sign-up + Metrics Collection

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    actor Athlete
    participant Screen as SignupScreen
    participant Actions as userActions.js
    participant Auth as Firebase Auth
    participant FS as Firestore
    participant AS as AsyncStorage
    participant Store as Redux store
    participant Modal as DataCollectModal

    rect rgb(46, 110, 70)
    Note over Athlete,Modal: Create the account and the athlete document
    Athlete->>Screen: first, last, email, password, accept ToS
    Screen->>Actions: dispatch startSignupWithEmail(...)
    Actions->>Auth: createUserWithEmailAndPassword
    Auth-->>Actions: user
    Actions->>FS: setDoc Users/uid with names, email, createdAt, empty pendingPermissions, coachList, disabledCoachList
    Actions->>AS: storeUID, storeFirstName, storeLastName, storeEmail
    Actions->>Actions: dispatch startUpdateProfile(first, last)
    Actions->>Auth: updateProfile displayName "First Last"
    end

    rect rgb(40, 85, 140)
    Note over Athlete,Modal: Collect body metrics
    Actions->>Store: userMetricsDataModalVisible(true, isFromSignUpScreen true)
    Actions->>Store: signupWithEmail(uuid, firstName, lastName, email)
    Store-->>Modal: visible = true
    Modal->>Athlete: gender, sport, DOB, height, weight sliders
    alt Save
        Athlete->>Modal: tap Save
        Modal->>Actions: dispatch startUpdateUserData(gender, dob, height, weight, sports)
        Actions->>FS: updateDoc Users/uid metrics
        Actions->>Store: updateUserMetricsData(metrics)
        Actions->>AS: storeMetrics(metrics)
    else Skip
        Athlete->>Modal: tap Skip
        Modal->>Actions: startUpdateUserData with empty values
    end
    Modal->>Store: userMetricsDataModalVisible(false, false)
    end
```

### Legend

- **Green** = account creation and the first write of `Users/uid`
- **Blue** = the metrics modal that opens straight after sign-up

---

## 3. Coach Invite -> Push Notification -> Permission Grant

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    actor Coach
    participant Dash as Coaching Dashboard
    participant FS as Firestore
    participant Fn as Cloud Function<br/>sendInvitationNotification
    participant Expo as Expo Push Service
    participant App as App.js
    participant Modal as NotificationPermissionsModal
    participant Actions as userActions.js
    actor Athlete

    rect rgb(95, 95, 95)
    Note over Coach,FS: External - the Coaching Dashboard repo
    Coach->>Dash: Invite athlete by email
    Dash->>FS: addDoc Invitations with athleteEmail, coachId, coach names
    end

    rect rgb(46, 110, 70)
    Note over FS,Expo: Server side - functions/index.js
    FS->>Fn: onCreate Invitations/invitationId
    Fn->>FS: Users where email == athleteEmail
    FS-->>Fn: athlete doc with expoPushToken
    Fn->>Expo: sendPushNotificationsAsync to token, data screen Home, showPermissionsModal true, coach id and names
    end

    rect rgb(40, 85, 140)
    Note over Expo,Athlete: Client side - App.js and the modal
    Expo-->>Athlete: Notification "Allow Coach X to Track Your Data?"
    Athlete->>App: Tap notification
    App->>App: handleNotificationResponse(response)
    App->>App: getUID() and screen == Home and showPermissionsModal
    App->>Modal: setRecentNotificationCoach, dispatch coachNotificationPermissionsModalVisible(true)
    Modal->>Actions: dispatch fetchPendingPermissions()
    Actions->>FS: getDoc Users/uid .pendingPermissions, then getDoc each coach ref
    FS-->>Actions: list of coachId, firstName, lastName, ref
    Modal->>Athlete: List of pending coaches
    alt Allow
        Athlete->>Modal: Allow
        Modal->>Actions: removeFromPendingPermissions(coach) - updateDoc arrayRemove
        Modal->>Actions: startAddToCoachAccess(coach) - updateDoc coachList arrayUnion
    else Deny
        Athlete->>Modal: Deny
        Modal->>Actions: removeFromPendingPermissions(coach)
    end
    Modal->>App: coachNotificationPermissionsModalVisible(false)
    end
```

### Legend

- **Gray** = the external Coaching Dashboard
- **Green** = the Cloud Function in this repo (`functions/index.js`)
- **Blue** = the mobile app handling the tap and the athlete's decision

The dashboard is expected to have already appended the coach's document reference to `Users/{uid}.pendingPermissions`; the push only wakes the athlete. If the app is not open, the same modal opens on the next launch via `checkPendingPermissions` (see [[06-Code]], section 4).

---

## 4. Health Tab: Date Range -> Firestore Query -> Chart

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    actor Athlete
    participant Toolbar as DateToolbar
    participant Store as Redux store (app)
    participant Screen as ViewHeartRateData
    participant Actions as userActions.js
    participant FS as Firestore
    participant Chart as HeartRateChart

    rect rgb(40, 85, 140)
    Athlete->>Toolbar: Pick start and end date (dateType period)
    Toolbar->>Store: dispatch updateHeartRateDateRange(start, end)
    Store-->>Screen: heartRateDateRangeData changed
    Screen->>Actions: queryHeartRateData(start, end)
    Actions->>FS: query Users/uid/HeartRateData where date between start and end
    FS-->>Actions: docs with date, heartRate
    Actions-->>Screen: array, or empty on error
    Screen->>Screen: min / max heartRate
    Screen->>Chart: dataArray = result
    Chart->>Athlete: LineChart of heartRate values
    Note over Athlete,Chart: ViewSleepData follows the same path with querySleepData, sleepDataDateRangeData and a CartesianChart of parsed stages
    end
```

### Legend

- **Blue** = the whole query path, which runs entirely in the app

Nothing in this repository ever writes these documents; the query returns `[]` for a fresh user. See [[Health_Data_Screens]].

---

## 5. Logout

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    actor Athlete
    participant Settings as SettingsScreen
    participant Actions as userActions.js
    participant FS as Firestore
    participant Auth as Firebase Auth
    participant AS as AsyncStorage
    participant Store as Redux store

    rect rgb(95, 95, 95)
    Athlete->>Settings: Tap Log out
    Settings->>Actions: dispatch startLogout()
    Actions->>Actions: dispatch clearPushTokenOnLogout()
    Actions->>FS: updateDoc Users/uid expoPushToken null
    Actions->>AS: clearToken()
    Actions->>Auth: signOut()
    Actions->>AS: clearUID, clearMetrics, clearFirstName, clearLastName, clearEmail
    Actions->>Store: logout()
    Actions->>Store: toastError "User logged out!"
    Store-->>Athlete: AppRouter renders AuthStack
    end
```

### Legend

- **Gray** = teardown: push token, Firebase session and AsyncStorage cache are all cleared

`deleteAccount()` follows the same shape but calls `user.delete()`, then `deleteDoc(Users/{uid})`, then `signOut()`. Health subcollections are not deleted (Firestore does not cascade).

---

## Key Relationships

| Relationship | Protocol | Purpose |
|-------------|----------|---------|
| App ↔ Firebase Auth | Firebase JS SDK | Sign-in, sign-up, persisted session, sign-out |
| App ↔ Firestore | Firebase JS SDK | `Users/{uid}` profile, metrics, coach refs, health queries |
| App ↔ AsyncStorage | Local key-value | `uid` and metrics cache that makes login sticky |
| Dashboard → Firestore → Cloud Function | `onCreate` trigger | A new invitation fires the push |
| Cloud Function → Expo Push → device | HTTPS | Wakes the athlete to grant coach access |

## Related Notes

- [[06-Code]]
- [[04-Component-Frontend]]
- [[05-Component-Backend]]
- [[Authentication]]
- [[Coach_Access_Permissions]]
- [[Push_Notifications_Client]]
- [[Cloud_Functions]]
