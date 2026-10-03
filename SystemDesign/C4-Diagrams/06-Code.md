# C4 Model - Level 4: Code Diagrams

Code-level detail for the parts of the system where structure matters most: the Firestore document schema, the Redux state shape, the boot / auth state machine in `App.js`, and the pending-permissions check that decides whether the coach modal opens.

## 1. Firestore Schema

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
erDiagram
    USERS {
        string uid PK "doc id = Firebase Auth uid"
        string firstName
        string lastName
        string email
        timestamp createdAt
        string gender "from DataCollectModal"
        timestamp dob
        number height "inches"
        number weight "lbs"
        string sports
        string expoPushToken "null after logout"
        reference_array pendingPermissions "refs to Users/coachId"
        reference_array coachList "refs to Users/coachId"
        reference_array disabledCoachList "written at signup"
        reference_array CoachListDisabled "read by fetchCoachAccess"
        string role "written by dashboard"
    }
    HEART_RATE_DATA {
        timestamp date
        number heartRate
    }
    SLEEP_DATA {
        timestamp startDate
        timestamp endDate
        string sleepValue "In Bed | Awake | Core | Rem | Deep"
    }
    HEART_RATE_DATA_HC {
        timestamp time
        number beatsPerMinute
    }
    SLEEP_DATA_HC {
        timestamp startTime
        timestamp endTime
        number stage "Health Connect stage code"
    }
    VO2_MAX_DATA_HX {
        string shape "read by dashboard only"
    }
    INVITATIONS {
        string id PK
        string athleteEmail
        string coachId
        string coachFirstName
        string coachLastName
    }

    USERS ||--o{ HEART_RATE_DATA : "app reads"
    USERS ||--o{ SLEEP_DATA : "app reads"
    USERS ||--o{ HEART_RATE_DATA_HC : "dashboard reads"
    USERS ||--o{ SLEEP_DATA_HC : "dashboard reads"
    USERS ||--o{ VO2_MAX_DATA_HX : "dashboard reads"
    USERS }o--o{ USERS : "pendingPermissions / coachList refs"
    INVITATIONS }o--|| USERS : "matched by email in Cloud Function"
```

### Legend

- **Entities** = Firestore collections and subcollections; `PK` is the document id
- **Relationship labels** = which system reads the subcollection (the app or the Coaching Dashboard)

Nothing in this repository writes to any of the five health subcollections. See [[Firestore_Schema]] for the full field reference.

---

## 2. Redux State Shape

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
classDiagram
    class RootState {
        user: UserState
        app: AppState
        device: DeviceState
        toast: ToastState
    }
    class UserState {
        uuid: string or null
        firstName: string or null
        lastName: string or null
        email: string or null
        authError: null
        userMetricsData: UserMetrics
        coachAccess: Coach[]
        coachAccessDisabled: Coach[]
        pendingPermissions: Coach[]
    }
    class UserMetrics {
        gender: string default NoData
        dob: Date
        height: number or NoData
        weight: number or NoData
        sports: string default NoData
    }
    class Coach {
        coachId: string
        firstName: string
        lastName: string
        ref: DocumentReference
        sharingEnabled: boolean optional
    }
    class AppState {
        darkTheme: boolean = false
        userMetricsDataModalVisible: boolean
        coachNotificationPermissionsModalVisible: boolean
        isFromSignUpScreen: boolean
        measurementSystem: imperial or metric
        activityRingsData: Record~Day,Rings~
        heartRateDateRangeData: DateRange
        sleepDataDateRangeData: DateRange
    }
    class Rings {
        ring1: number
        ring2: number
        ring3: number
    }
    class DateRange {
        startDate: Date
        endDate: Date
    }
    class DeviceState {
        devicesData: DummyDevice[] hardcoded
    }
    class ToastState {
        show: boolean
        message: string
        error: boolean
    }
    RootState --> UserState
    RootState --> AppState
    RootState --> DeviceState
    RootState --> ToastState
    UserState --> UserMetrics
    UserState --> Coach
    AppState --> Rings
    AppState --> DateRange
```

### Legend

- **Classes** = Redux slices and the shapes stored in them; arrows point from a slice to the types it holds

Reducers: `src/reducers/{userReducer,appReducer,deviceReducer,toastReducer}.js`; combined in `src/store.js` with `redux-thunk`. See [[Redux_State]].

---

## 3. Boot / Auth State Machine (`App.js`)

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
stateDiagram-v2
    [*] --> Loading : App mounts (isLoading = true)
    Loading --> Loading : useAppFonts() + onAuthStateChanged subscribe

    Loading --> auth : onAuthStateChanged fires

    state auth {
        [*] --> HasUser
        [*] --> NoUser
        HasUser --> CheckStoredUID : getUID() from AsyncStorage
        CheckStoredUID --> Restored : uid found, dispatch restoreUUID()
        CheckStoredUID --> NotRestored : uid missing, stay logged out
        Restored --> Metrics
        NotRestored --> Metrics
        NoUser --> Metrics : checkUID(), checkMetrics()
        Metrics --> PendingCheck : cached metrics if present, else startLoadUserData()
        PendingCheck --> [*] : checkPendingPermissions(user)
    }

    auth --> Ready : setLoading(false), then startRegisterPushToken()

    state Ready {
        [*] --> Router
        Router --> AuthStack : state.user.uuid == null
        Router --> MainTabs : state.user.uuid != null
        MainTabs --> CoachModal : pendingPermissions.length > 0 or notification tap
        CoachModal --> MainTabs : accept / deny / close
    }
```

### Legend

- **States** = what `App.js` renders while booting; `auth` and `Ready` are composite states
- **Transitions** = the event or check that moves the app on

Notes:
- The store is created once at module scope (`const store = configureStore()`), so Redux state survives re-renders but not app restarts; AsyncStorage is what makes login sticky.
- Firebase Auth has its own persisted session (`getReactNativePersistence`), so `onAuthStateChanged` normally fires with a user on cold start; the app still requires the AsyncStorage `uid` to consider itself logged in.

---

## 4. Pending-Permissions Check

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    start["checkPendingPermissions(user)<br/>called after auth + on isLoggedIn change"]
    uid{"getUID() and user?"}
    src{"user.pendingPermissions<br/>present on the auth object?"}
    fetch["fetchPendingPermissionsList(uid)<br/>getDoc Users/uid .pendingPermissions"]
    len{"length > 0 ?"}
    open["dispatch coachNotificationPermissionsModalVisible(true)<br/>NotificationPermissionsModal opens"]
    skip["log and do nothing"]

    start --> uid
    uid -- no --> skip
    uid -- yes --> src
    src -- yes --> len
    src -- no --> fetch --> len
    len -- yes --> open
    len -- no --> skip

    style start fill:#438DD5,color:#fff,stroke:#3579B8
    style fetch fill:#438DD5,color:#fff,stroke:#3579B8
    style open fill:#1168BD,color:#fff,stroke:#0E5CA4
    style skip fill:#999,color:#fff,stroke:#888
```

### Legend

- **Blue** = steps the app runs
- **Dark blue** = the outcome the athlete sees (the modal opens)
- **Gray** = the silent no-op path

`NotificationPermissionsModal` then dispatches `fetchPendingPermissions()` itself to resolve each coach reference into `{coachId, firstName, lastName, ref}` before rendering the list. See [[Coach_Access_Permissions]].

---

## Key Relationships

| Relationship | Protocol | Purpose |
|-------------|----------|---------|
| `Users/{uid}` → coach `Users` docs | Firestore DocumentReference arrays | `pendingPermissions`, `coachList`, disabled lists |
| `Invitations` → `Users` | Email match in the Cloud Function | Finds the athlete to push to |
| Redux `user` slice ↔ AsyncStorage | `src/utils/localStorage.js` | Survives app restarts |
| `App.js` boot ↔ Firebase Auth | `onAuthStateChanged` | Restores the session before routing |

## Related Notes

- [[Firestore_Schema]]
- [[Redux_State]]
- [[Authentication]]
- [[Coach_Access_Permissions]]
- [[07-Dynamic-Diagrams]]
