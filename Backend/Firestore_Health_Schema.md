# Firestore Health Schema (DailyStats + AIFeedback)

*Created 2026-10-21. This is the schema [[T-009_Save_Daily_Stats_To_Firestore]] builds, phase by phase. It's the contract the Smart Clothing App writes (Android and iOS) and the Coaching Dashboard reads, until the Boracle platform replaces Firestore. The old-vs-new comparison is in [[Firestore_Schema_Proposal]]; the old collections are in [[Firestore_Schema]].*

**Status:** designed, not built yet. Both collections are empty until T-009 P3 ships. Real data only, never fake or seed data.

---

## Overview

```
Users/{uid}                       existing: profile, role, coachList, authorizedAthletes
 ├── DailyStats/{YYYY-MM-DD}      NEW  one doc per finished day      writer: the app
 └── AIFeedback/{YYYY-MM-DD}      NEW  that day's Claude feedback    writer: the app
```

`Users/{coachUid}/TeamFeedback/...` (the coach's team assessment) is designed and written by the senior design team in [[T-012_Team_Insights_AI_Feedback_PoC]]. It's not part of this schema.

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    athlete["<b>Athlete</b><br/>Pixel Watch or Apple Watch"]
    coach["<b>Coach</b>"]

    subgraph writers["Writers - Smart Clothing App"]
        android["<b>Android</b><br/>Fitbit API until 10/30,<br/>then Health Connect (T-001)"]
        ios["<b>iOS</b><br/>HealthKit (T-010)"]
        store["<b>healthStore.js</b><br/>the only code that writes<br/>these collections"]
    end

    subgraph fs["Firestore - Users/uid"]
        users["<b>Users/uid</b><br/>existing profile and roles"]
        daily["<b>DailyStats/YYYY-MM-DD</b><br/>one doc per finished day"]
        fb["<b>AIFeedback/YYYY-MM-DD</b><br/>that day's Claude feedback"]
    end

    dash["<b>Coaching Dashboard</b><br/>Insights API, firebase-admin<br/>T-011 athlete page, T-012 Team Insights"]

    athlete -->|"uses"| android
    athlete -->|"uses"| ios
    android --> store
    ios --> store
    store -->|"1 write per finished day"| daily
    store -->|"1 write per Get feedback<br/>or Regenerate"| fb
    users -->|"contains"| daily
    users -->|"contains"| fb
    daily -.->|"same date"| fb
    coach -->|"uses"| dash
    athlete -->|"uses"| dash
    dash -->|"reads"| daily
    dash -->|"reads"| fb

    style athlete fill:#08427B,color:#fff,stroke:#073B6F
    style coach fill:#08427B,color:#fff,stroke:#073B6F
    style android fill:#438DD5,color:#fff,stroke:#3579B8,stroke-dasharray: 5 5
    style ios fill:#438DD5,color:#fff,stroke:#3579B8,stroke-dasharray: 5 5
    style store fill:#438DD5,color:#fff,stroke:#3579B8,stroke-dasharray: 5 5
    style users fill:#438DD5,color:#fff,stroke:#3579B8
    style daily fill:#85BBF0,color:#000,stroke:#5D9FD6,stroke-dasharray: 5 5
    style fb fill:#85BBF0,color:#000,stroke:#5D9FD6,stroke-dasharray: 5 5
    style dash fill:#999,color:#fff,stroke:#888
    style writers fill:#E8E8E8,color:#333,stroke:#BBB
    style fs fill:#E8E8E8,color:#333,stroke:#BBB
```

### Legend

- **Dark blue** = people
- **Blue** = the app (**dashed** = new or changed by T-009) and the existing `Users` document
- **Light blue dashed** = the two new collections
- **Gray** = the Coaching Dashboard (separate repo), which only reads
- **Light gray boxes** = boundaries
- **Solid arrow** = uses, writes, contains or reads; **dashed arrow** = `AIFeedback` shares its date with that day's `DailyStats`

---

## One `DailyStats` document

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    doc["<b>Users/uid/DailyStats/2026-10-22</b><br/>one finished day, overwritten on re-save"]

    meta["<b>Metadata</b><br/>date, schemaVersion 1,<br/>source fitbit healthkit health_connect,<br/>device, timezone, syncedAt"]
    steps["<b>steps</b><br/>day total"]
    sleep["<b>sleep</b><br/>totalMinutes, efficiency,<br/>deep, light, rem, wake minutes,<br/>bedTime, wakeTime"]
    stages["<b>sleep.stages</b><br/>about 50 entries a night:<br/>start, seconds, level<br/>deep light rem wake"]
    heart["<b>heart</b><br/>restingHeartRate,<br/>previousRestingHeartRate,<br/>zones fatBurn cardio peak minutes"]
    hr5["<b>heart.hr5min</b><br/>288 values: avg bpm<br/>for each 5 minutes from midnight"]
    workouts["<b>workouts</b> - one entry per session<br/>id, type Run Weights Walk Workout,<br/>startTime, durationMin, calories,<br/>distanceKm, paceMinPerKm, steps,<br/>avgHeartRate, hasGps, zones"]

    doc --> meta
    doc --> steps
    doc --> sleep
    doc --> heart
    doc --> workouts
    sleep --> stages
    heart --> hr5

    style doc fill:#438DD5,color:#fff,stroke:#3579B8
    style meta fill:#85BBF0,color:#000,stroke:#5D9FD6
    style steps fill:#85BBF0,color:#000,stroke:#5D9FD6
    style sleep fill:#85BBF0,color:#000,stroke:#5D9FD6
    style stages fill:#85BBF0,color:#000,stroke:#5D9FD6
    style heart fill:#85BBF0,color:#000,stroke:#5D9FD6
    style hr5 fill:#85BBF0,color:#000,stroke:#5D9FD6
    style workouts fill:#85BBF0,color:#000,stroke:#5D9FD6
```

### Legend

- **Blue** = the document
- **Light blue** = its fields (maps and arrays)
- **Arrow** = contains

A run's stats card (duration, energy burned, distance, pace, steps, average HR) reads one `workouts[]` entry. There are no day activity totals (`activity` was cut on 2026-10-21; [[T-008_Daily_Activity_Stats]] may add them later).

---

## One `AIFeedback` document

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart TD
    doc["<b>Users/uid/AIFeedback/2026-10-22</b><br/>overwritten by Regenerate"]

    meta["<b>Metadata</b><br/>date, schemaVersion 1,<br/>dataSource, createdAt"]
    reply["<b>Claude's reply</b><br/>headline - the decision, max 8 words<br/>keySignals - up to 2, with numbers<br/>focus - up to 2, for the next session"]
    cost["<b>Cost tracking</b><br/>model that answered,<br/>inputTokens, outputTokens"]
    regen["<b>regeneratedCount</b><br/>0 on first generate,<br/>+1 per Regenerate"]

    doc --> meta
    doc --> reply
    doc --> cost
    doc --> regen

    style doc fill:#438DD5,color:#fff,stroke:#3579B8
    style meta fill:#85BBF0,color:#000,stroke:#5D9FD6
    style reply fill:#85BBF0,color:#000,stroke:#5D9FD6
    style cost fill:#85BBF0,color:#000,stroke:#5D9FD6
    style regen fill:#85BBF0,color:#000,stroke:#5D9FD6
```

### Legend

- **Blue** = the document
- **Light blue** = groups of fields
- **Arrow** = contains

---

## When the app writes

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
flowchart LR
    open["<b>App opens or<br/>returns to foreground</b><br/>P5"]
    day["<b>Athlete opens a<br/>finished day</b><br/>P3"]
    gen["<b>Get feedback or<br/>Regenerate succeeds</b><br/>P4"]

    saveDay["<b>saveDailyStats</b><br/>yesterday plus missed days<br/>never today"]
    saveFb["<b>saveAIFeedback</b><br/>not for NO_DATA, errors<br/>or cache hits"]

    daily["<b>DailyStats/date</b>"]
    fb["<b>AIFeedback/date</b>"]

    open --> saveDay
    day --> saveDay
    gen --> saveFb
    saveDay --> daily
    saveFb --> fb

    style open fill:#08427B,color:#fff,stroke:#073B6F
    style day fill:#08427B,color:#fff,stroke:#073B6F
    style gen fill:#08427B,color:#fff,stroke:#073B6F
    style saveDay fill:#438DD5,color:#fff,stroke:#3579B8,stroke-dasharray: 5 5
    style saveFb fill:#438DD5,color:#fff,stroke:#3579B8,stroke-dasharray: 5 5
    style daily fill:#85BBF0,color:#000,stroke:#5D9FD6,stroke-dasharray: 5 5
    style fb fill:#85BBF0,color:#000,stroke:#5D9FD6,stroke-dasharray: 5 5
```

### Legend

- **Dark blue** = what triggers a save (with the T-009 phase that adds it)
- **Blue dashed** = `healthStore.js` functions
- **Light blue dashed** = the documents
- **Arrow** = leads to

A failed write is logged and never breaks a screen. Re-saving the same day overwrites the same document (id = date), so there are never duplicates.

---

## Field reference

### `Users/{uid}/DailyStats/{YYYY-MM-DD}`

| Field | Type | Content |
|-------|------|---------|
| `date` | string | The athlete's **local** date, same as the id |
| `schemaVersion` | number | `1`; bumped on any breaking change |
| `source` | string | `"fitbit"`, `"healthkit"` or `"health_connect"` |
| `device` | string or null | Watch model, if known |
| `timezone` | string | IANA zone of the phone when saved |
| `syncedAt` | timestamp | `serverTimestamp()` |
| `steps` | number or null | Day total |
| `sleep` | map or null | `totalMinutes`, `efficiency`, `deep`, `light`, `rem`, `wake`, `bedTime`, `wakeTime`, `stages` |
| `sleep.stages` | array | `[ { start, seconds, level } ]`; `level` is `deep`, `light`, `rem`, `wake` (classic nights: `asleep`, `restless`, `awake`) |
| `heart` | map | `restingHeartRate`, `previousRestingHeartRate`, `zones { fatBurn, cardio, peak }`, `hr5min` |
| `heart.hr5min` | 288 × (number or null), or null | Average bpm per 5-minute slot from local midnight |
| `workouts` | array | `[ { id, type, startTime, durationMin, avgHeartRate, calories, steps, distanceKm, paceMinPerKm, hasGps, zones } ]` |

### `Users/{uid}/AIFeedback/{YYYY-MM-DD}`

| Field | Type | Content |
|-------|------|---------|
| `date` | string | Same as the id |
| `schemaVersion` | number | `1` |
| `headline` | string | The decision for the next session |
| `keySignals` | string[] | Up to 2 pieces of evidence, each with a number |
| `focus` | string[] | Up to 2 constraints for the next session |
| `model` | string | The model that actually answered (can differ after a fallback) |
| `inputTokens`, `outputTokens` | number | For cost: tokens × the model's price |
| `dataSource` | string | `source` of the day the feedback used |
| `regeneratedCount` | number | How many times Regenerate was tapped |
| `createdAt` | timestamp | `serverTimestamp()` |

The plain-English "what each field is for" section is in [[T-009_Save_Daily_Stats_To_Firestore]].

### Rules for every field

- Missing values are **`null`, never `0`**.
- Times are **local** ISO strings without an offset; `timezone` says which zone.
- No raw vendor payloads, no GPS routes.
- Only finished days; today is never saved.

### What is not stored

| Data | Why not |
|------|---------|
| Per-minute heart rate as documents | About 1,440 writes per athlete per day. `hr5min` keeps the day's HR line in one write |
| Sleep stage segments as documents | About 30-80 writes per night. They're in `sleep.stages` |
| Day activity totals | Cut 2026-10-21; per-workout stats are in `workouts[]` |
| GPS routes | Large and sensitive; not needed by the dashboard yet |
| Today's unfinished day | It would be overwritten by the full day |

---

## Write budget

| Per athlete per day | Writes |
|---------------------|--------|
| `DailyStats` | 1 |
| `AIFeedback` (plus the occasional Regenerate) | 1-3 |
| **Total** | **about 2-4** |

60 athletes is about 120-240 writes a day, about 1% of the 20,000-write free tier.

## Security (T-009 P6)

`DailyStats` and `AIFeedback`: read and write only by their owner (`request.auth.uid == uid`). The dashboard reads through its API with `firebase-admin`, which bypasses rules. The catch-all rule is narrowed at the same time, agreed with the dashboard team ([[Firestore_Rules]]).

## Related Notes

- [[T-009_Save_Daily_Stats_To_Firestore]]
- [[Firestore_Schema_Proposal]]
- [[Firestore_Schema]]
- [[Firestore_Rules]]
- [[T-011_Dashboard_Athlete_Health_Page]]
- [[T-012_Team_Insights_AI_Feedback_PoC]]
- [[T-010_iOS_Apple_Watch_AI_Feedback_PoC]]
- [[T-001_Android_Health_Connect]]
