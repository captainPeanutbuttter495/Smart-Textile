---
id: T-009
title: "Firestore health schema: save each athlete's day (DailyStats) and AI feedback (AIFeedback), one document per day, every day (phases P0-P7)"
area: Data
priority: High
status: In Progress (P0-P5 done)
assignee: Matthew Garcia
github_issue: —
created: 2026-10-14
updated: 2026-10-26
---

# T-009: Firestore health schema: save each athlete's day and AI feedback, every day

*Merged on 2026-10-17: this ticket includes [[T-007_Save_AI_Feedback_To_Firestore]]. Slimmed and split into phases on 2026-10-21 (see "Decisions"). It defines the **Firestore contract** that the Smart Clothing App writes (Android and iOS), and that the Coaching Dashboard reads ([[T-011_Dashboard_Athlete_Health_Page]], [[T-012_Team_Insights_AI_Feedback_PoC]]) until the Boracle platform takes over.*

**Diagrams and the full field reference:** [[Firestore_Health_Schema]] (in `Backend/`).

**How this ticket is worked:**
- Each phase has its own ticket in this folder (`Tickets/Health_Schema/T-009-Px_*.md`), with its scope, checklist and diagrams. Its write-up sections are filled in when the phase ends.
- This master ticket is the checklist. It's updated at the end of each phase chat, once the lead has checked the phase by hand.
- One phase per chat. Start each chat by reading the "Progress checklist" below and that phase's ticket.
- **Branch:** `feature/health-schema`, created from `feature/insights-service`. That branch counts as the main branch for now; nothing merges to `main`.

## Summary

Nothing from a watch is stored anywhere today. The app fetches each day through `insights-service` and keeps it in memory, and the AI feedback lives only in the phone's AsyncStorage cache.

This ticket saves two things for each athlete, **every finished day from now on**:
- **`DailyStats/{date}`**: one document per finished day, with steps, sleep (with its stage timeline), heart rate (with a 5-minute timeline), VO2 max and workouts (each with its own stats).
- **`AIFeedback/{date}`**: Claude's feedback for that day, with the model and token counts.

Both are written from **platform-agnostic** code, so the Android app (Pixel Watch, `source: "fitbit"`, later `"health_connect"`) and the iOS app (Apple Watch, [[T-010_iOS_Apple_Watch_AI_Feedback_PoC]], `source: "healthkit"`) produce identical documents.

There is **no backfill** of past weeks: the goal is a day-by-day record going forward, so the athlete (and later the dashboard) can look back at any day and its feedback.

## Why

- **Dr. Jiang (2026-10-13):** save the lead's real data to Firebase so the senior design team can build the athlete view and Team Insights on real data. The dashboard is mostly hard-coded today, and its old data is unlabelled (some real, some fake).
- **Real data only (lead, 2026-10-17):** no fake or seed data goes into Firebase. Every document carries `source` and `device`, so its origin is always known.
- **Every day, not a one-off upload (lead, 2026-10-21):** the next steps are feedback for each day and looking back at any day, so the save has to run daily. The last two weeks have no workouts, so a backfill would add little.
- **The Firestore free tier allows 20,000 writes a day.** Carlos raised the concern that three people uploading every day could hit it. The schema is **one document per athlete per day** for exactly this reason (see "Write budget").
- **One contract for every watch and for Boracle later.** The dashboard should never care which watch or app wrote the data, and the shape maps onto a Postgres table when the Boracle platform replaces Firestore (Dr. Jiang, 2026-10-13).
- Replaces the "Schema divergence (decision needed)" section of [[Firestore_Schema]] for the new data. The old collections aren't touched here; they are archived and cleaned up in [[T-013_Firestore_Cleanup_Fresh_Start]]. The old-vs-new comparison is in [[Firestore_Schema_Proposal]].

### Fitbit cutoff (2026-10-30)

The Fitbit Web API stops on 2026-10-30. Daily saving through it only works until then, so **P1-P5 should be live as early as possible (target about 2026-10-25)** to start collecting real days. After the cutoff, the same `saveDailyStats` keeps working once [[T-001_Android_Health_Connect]] builds the day on the phone with `source: "health_connect"`; nothing in this ticket's code changes for that. Days between the cutoff and T-001 landing will be missing, and that's accepted.

---

## Decisions (lead, 2026-10-21)

| Decision | Why |
|----------|-----|
| **No `activity` field** (day totals: calories out, active minutes, floors) | The stats card the lead cares about is **per workout**: duration, energy burned, distance, pace, steps and average heart rate for that run. Those already live in each `workouts[]` entry (the same fields `WorkoutStats.jsx` shows). Day totals can be added later by [[T-008_Daily_Activity_Stats]] as a new field without breaking anything. |
| **No `promptVersion`** in `AIFeedback` | Not needed for now. Can be added later. |
| **`TeamFeedback` belongs to T-012** | It's the coach-side team assessment, designed and written by the senior design team in [[T-012_Team_Insights_AI_Feedback_PoC]]. This ticket doesn't define its shape or rules. |
| **`heart.hr5min` at 5-minute resolution** | See "Why 5 minutes" below. |
| **No "Upload last 7 days"** | Replaced by daily auto-save (P5). |
| **Add `vo2Max`** (added 2026-10-21, after the call with Carlos) | The coach's Athlete Info page has a VO2 max card, which today reads fake `VO2MaxDataHX` docs. One small field per day, no extra writes. Needs the Fitbit `cardio_fitness` scope (one re-login). |

---

## The schema

```
Users/{uid}                       existing: profile, role, coachList, authorizedAthletes
 ├── DailyStats/{YYYY-MM-DD}      NEW  athlete's finished day        writer: the app
 └── AIFeedback/{YYYY-MM-DD}      NEW  athlete's Claude feedback     writer: the app
```

*(`Users/{coachUid}/TeamFeedback/...` is T-012's, not this ticket's.)*

The diagrams (overview, one `DailyStats` document, one `AIFeedback` document) are in [[Firestore_Health_Schema]].

### `Users/{uid}/DailyStats/{YYYY-MM-DD}`

The document id is the athlete's **local** date. A re-save **overwrites** the document, so it's idempotent. Only **finished** days are written (never today), so a half day never overwrites a full one.

| Field | Type | Content | Example |
|-------|------|---------|---------|
| `date` | string | Same as the id | `"2026-10-22"` |
| `schemaVersion` | number | Bumped on any breaking change | `1` |
| `source` | string | `"fitbit"`, `"healthkit"` or `"health_connect"` | `"fitbit"` |
| `device` | string or null | Watch model, if known | `"Pixel Watch 3"` |
| `timezone` | string | IANA zone of the phone when saved | `"America/Los_Angeles"` |
| `syncedAt` | timestamp | `serverTimestamp()` | |
| `steps` | number or null | Day total | `9842` |
| `sleep` | map or null | `totalMinutes`, `efficiency`, `deep`, `light`, `rem`, `wake`, `bedTime`, `wakeTime`, **`stages`** | |
| `sleep.stages` | array | `[ { start, seconds, level } ]`, `level` one of `"deep"`, `"light"`, `"rem"`, `"wake"` (Fitbit "classic" nights: `"asleep"`, `"restless"`, `"awake"`). Same names the app's hypnogram uses | `{ start: "2026-10-22T01:12:30", seconds: 1800, level: "deep" }` |
| `heart` | map | `restingHeartRate`, `previousRestingHeartRate`, `zones { fatBurn, cardio, peak }` (minutes), **`hr5min`** | |
| `vo2Max` | map or null | `{ low, high }` in ml/kg/min. Fitbit usually gives a range (`"44-48"` -> `low: 44, high: 48`); when it gives one number, `low` and `high` are the same. `null` on days with no estimate | `{ low: 44, high: 48 }` |
| `heart.hr5min` | array of 288 (number or null), or null | Average bpm for each 5-minute slot from local midnight (index 0 = 00:00-00:05). `null` entries where the watch has no reading; the whole field is `null` if the intraday call failed | `[58, 57, null, ...]` |
| `workouts` | array | `[ { id, type, startTime, durationMin, avgHeartRate, calories, steps, distanceKm, paceMinPerKm, hasGps, zones } ]`. `type` is Run, Weights, Walk or Workout. Distances in km (the app converts to miles for display) | |

Rules for every field:
- Missing values are **`null`, never `0`**, so "no data" and "zero" stay different.
- Times are **local** ISO strings without an offset (`2026-10-22T20:53:00`), as Fitbit returns them. `timezone` says which zone.
- No raw vendor payloads and no GPS routes (routes stay on-device and in Fitbit/HealthKit).

**Size check:** 288 numbers plus about 50 stage entries plus a few workouts is well under 10 KB, against Firestore's 1 MiB document limit.

### `Users/{uid}/AIFeedback/{YYYY-MM-DD}`

Written right after a successful **Get feedback** or **Regenerate** in the app. Regenerate overwrites it.

| Field | Type | Content |
|-------|------|---------|
| `date` | string | Same as the id |
| `schemaVersion` | number | `1` |
| `headline` | string | From the service response |
| `keySignals` | string[] | |
| `focus` | string[] | |
| `model` | string | The model that actually answered, from the service's `usage.model` (for example `"claude-opus-5-5"`) |
| `inputTokens`, `outputTokens` | number | From the service's `usage` |
| `dataSource` | string | `source` of the day the feedback was based on |
| `regeneratedCount` | number | 0 on the first generate, then `increment(1)` |
| `createdAt` | timestamp | `serverTimestamp()` |

No write for `NO_DATA`, for errors, or when the phone cache answers.

---

## What each field is for (plain English)

**DailyStats**
- **Metadata** (`date`, `schemaVersion`, `source`, `device`, `timezone`, `syncedAt`): which day, which version of this shape, which watch and pipeline wrote it, and when. `schemaVersion` lets readers handle an old shape after a breaking change.
- **`steps`**: daily movement total, for the steps ring and the dashboard.
- **`sleep`**: totals for the night that ended on this date, plus **`stages`**, the stage-by-stage timeline the hypnogram draws.
- **`heart`**:
  - `restingHeartRate`: the watch's resting HR estimate for the day.
  - `previousRestingHeartRate`: yesterday's, because the coach prompt leads with a change of more than 5 bpm.
  - `zones`: minutes in Fat Burn, Cardio and Peak across the whole day.
  - **`hr5min`**: the day's heart-rate line, one average every 5 minutes.
- **`vo2Max`**: the watch's cardio fitness estimate (how much oxygen the body can use during hard exercise), most useful for runners. It changes slowly, so one value per day is plenty. Stored as a range because that's how Fitbit reports it.
- **`workouts`**: one entry per session with its own stats. For a run that's duration, energy burned, distance, pace, steps, average HR and zone minutes: exactly what the workout stats card shows.

**Why 5 minutes for `hr5min`?** Hourly (24 points) flattens the day: a 30-minute run averages in with 30 minutes of sitting and the spike disappears. Every 5 minutes (288 points) shows when a workout happened, how hard it was and how fast HR came back down. The resolution is **not** a write-cost question: the whole array is inside the day's one document, so 24, 288 or 1,440 values are all one write. What was rejected was per-minute readings *as separate documents* (about 1,440 writes per athlete per day). 5 minutes is enough for charts and keeps the payload small when the dashboard loads many athletes and days.

**AIFeedback**
- **`headline`, `keySignals`, `focus`**: the three parts of Claude's reply, set by the prompt in `insights-service/src/lib/coach.js`.
  - `headline`: the decision for the next session, at most 8 words ("Back off next session after short sleep").
  - `keySignals`: up to 2 pieces of evidence, each with a number from the data ("5h31m sleep, not enough to recover").
  - `focus`: up to 2 constraints for the next session.
  - Saving them lets the athlete look back at any day's feedback, and lets the dashboard show it **without a new Claude call**.
- **`model`**: which Claude model actually wrote the reply. The service allows a fallback model when the main one is unavailable, so it can differ from the configured one. Needed because prices differ by model, and to compare feedback quality between models.
- **`inputTokens`, `outputTokens`**: cost. Cost = input tokens × the model's input price + output tokens × its output price. These real counts feed the cost estimates (T-003 P5, T-012 P4).
- **`dataSource`**: which pipeline fed the day (`fitbit`, `healthkit`, `health_connect`), without opening `DailyStats`.
- **`regeneratedCount`**: how many times the athlete tapped Regenerate. Regenerate overwrites the document, so this counter is the only record that it happened. Each regenerate is a paid call, and a high count suggests the feedback wasn't trusted.

### What is not stored

| Data | Why not |
|------|---------|
| Per-minute or per-second heart rate as documents (`HeartRateDataHC` style) | About 1,440 writes per athlete per day: 60 athletes would be about 86,400 writes, over the free tier. `hr5min` covers charts and trends in one write |
| Sleep stage segments as separate documents (`SleepDataHC` style) | About 30-80 writes per night. They live in `sleep.stages` instead |
| Day activity totals (`activity`) | Cut on 2026-10-21; per-workout stats are in `workouts[]`. T-008 may add them later |
| GPS routes | Large, sensitive (they show where people live and train), and not needed by the dashboard yet |
| Today's unfinished day | It would be overwritten by the full day anyway |

---

## Write budget

| Per athlete per day | Writes |
|---------------------|--------|
| `DailyStats` (one finished day) | 1 |
| `AIFeedback` (one generate, plus the occasional Regenerate) | 1-3 |
| **Total** | **about 2-4** |

| Athletes | Writes per day | Free tier (20,000/day) |
|----------|----------------|------------------------|
| 3 (PoC) | about 6-12 | well under |
| 60 | about 120-240 | well under (about 1%) |

The daily auto-save (P5) re-saving a day that was already saved costs one extra write; it only does that for days since the last saved date. Reads on the dashboard side are sized in T-011 and T-012.

---

## Scope by phase

The full scope, checklist, lead checks and diagrams for each phase are in its own ticket.

| Phase | Ticket | In one line |
|-------|--------|-------------|
| P0 | [[T-009-P0_Schema_And_Docs]] | Field review, decisions, this ticket, [[Firestore_Health_Schema]] (done) |
| P1 | [[T-009-P1_Service_HR5min_VO2Max]] | `insights-service`: `heart.hr5min` (288 or null) and `vo2Max { low, high }` from optional Fitbit calls; `cardio_fitness` scope |
| P2 | [[T-009-P2_App_Save_Module]] | `src/services/healthStore.js`: `toDailyStatsDoc`, `saveDailyStats`, `saveAIFeedback`, not wired up yet |
| P3 | [[T-009-P3_Save_The_Day]] | Save a finished day after it loads on DailySummary; never today |
| P4 | [[T-009-P4_Save_The_Feedback]] | Save `AIFeedback` after Get feedback / Regenerate, with `usage` |
| P5 | [[T-009-P5_Daily_Auto_Save]] | Save yesterday and missed days on app open; live before 2026-10-30 |
| P6 | [[T-009-P6_Security_Rules]] | Owner-only rules for both collections; narrow the catch-all |
| P7 | [[T-009-P7_Wrap_Up_Handoff]] | [[Firestore_Schema]] update, handoff message, T-011 to GitHub Issues |

### iOS (with [[T-010_iOS_Apple_Watch_AI_Feedback_PoC]])
- `healthKitDay.js` also builds `heart.hr5min` from the HealthKit heart-rate samples (average per 5-minute slot) and `vo2Max` from HealthKit's VO2 max samples (latest of the day, `low` = `high`), and iOS calls the same `healthStore.js` functions with `source: "healthkit"`. No iOS-specific schema. Done by Carlos under T-010, not in these phases.

---

## Progress checklist

### Phase status

| Phase | Status | Branch / PR | Write-up |
|-------|--------|-------------|----------|
| P0 Schema + docs | Done (2026-10-21) | — (docs only) | [[T-009-P0_Schema_And_Docs]] |
| P1 Service: 5-min heart rate + VO2 max | Done (2026-10-25) | `feature/health-schema` | [[T-009-P1_Service_HR5min_VO2Max]] |
| P2 App: save module | Done (2026-10-25) | `feature/health-schema` | [[T-009-P2_App_Save_Module]] |
| P3 Save the day | Done (2026-10-25) | `feature/health-schema` | [[T-009-P3_Save_The_Day]] |
| P4 Save the feedback | Done (2026-10-26) | `feature/health-schema` | [[T-009-P4_Save_The_Feedback]] |
| P5 Daily auto-save | Done (2026-10-26), skipped-day check open | `feature/health-schema` | [[T-009-P5_Daily_Auto_Save]] |
| P6 Security rules | Deferred (2026-10-26): before other athletes' data or release; agree with the dashboard team first | — | [[T-009-P6_Security_Rules]] |
| P7 Wrap-up + handoff | Not started | — | [[T-009-P7_Wrap_Up_Handoff]] |

### P0 - Schema + docs
- [x] Field-by-field review; `activity`, `promptVersion`, `TeamFeedback` and the 7-day upload cut
- [x] Ticket rewritten with phases and checklist, moved to `Tickets/Health_Schema/`
- [x] [[Firestore_Health_Schema]] created; [[Firestore_Schema_Proposal]] updated
- [x] Senior design handoff drafted (not sent)
- [x] `vo2Max` added to the schema (after the 2026-10-21 call with Carlos)
- [x] One ticket per phase (T-009-P0 to P7) in `Health_Schema/`

### P1 - Service: 5-min heart rate + VO2 max
- [x] Feature branch `feature/health-schema` created from `feature/insights-service`
- [x] `getHeartDay5min` in `fitbit-api.js` + tests with mocked HTTP
- [x] `heart.hr5min` (288 or null) in `normalizeDay.js` + tests (full day, gaps)
- [x] `fetchDayRaw` calls it; a failed intraday call leaves `hr5min: null`
- [x] `getVo2Max` + `vo2Max: { low, high }` parsing (range, single value, empty, 403) + tests
- [x] `cardio_fitness` added to `FITBIT_SCOPES`
- [x] Lead reconnects Fitbit (2026-10-25)
- [x] `npm test` green in `insights-service/` (14 suites, 223 tests, 2026-10-21)
- [x] Lead: day endpoint for 2026-10-24 shows `hr5min` and `vo2Max { low, high }`
- [x] Write-up

### P2 - App: save module
- [x] `src/services/healthStore.js`: `toDailyStatsDoc`, `saveDailyStats`, `saveAIFeedback`
- [x] Unit tests (Firestore mocked), including nulls not zeros and the `levels` -> `stages` rename (11 tests)
- [x] `npm test` green in `SmartClothingApp/` (32 suites, 290 tests)
- [x] Write-up

### P3 - Save the day
- [x] `startLoadFitbitDay` saves after a successful load; today skipped
- [x] Failed write only logs + test
- [x] `npm test` green in `SmartClothingApp/` (32 suites, 295 tests)
- [x] Lead: yesterday's `DailyStats` doc in the console; no doc for today (2026-10-25; refresh overwrites, only `syncedAt` changes)
- [x] Write-up

### P4 - Save the feedback
- [x] `usage` kept; `saveAIFeedback` after `setCachedFeedback`
- [x] No write for `NO_DATA`, errors, cache hits + tests
- [x] `npm test` green in `SmartClothingApp/` (32 suites, 302 tests)
- [x] Lead: `AIFeedback` doc with model and tokens; Regenerate bumps `regeneratedCount` (2026-10-26, 2026-10-25 doc: 1970 in / 115 out, then count 1)
- [x] Write-up

### P5 - Daily auto-save
- [x] Save on app open / foreground: yesterday plus missed days since the last save (capped at 3)
- [x] Last saved date in AsyncStorage + tests
- [x] `npm test` green in `SmartClothingApp/` (32 suites, 321 tests)
- [x] Lead: doc written on open without opening DailySummary; no rewrite on foreground (2026-10-26, 2026-10-25 doc deleted then recreated)
- [ ] Lead: skipped day caught up on the next open
- [x] Live before 2026-10-30 (pushed 2026-10-26)
- [x] Write-up

### P6 - Security rules
- [ ] Owner-only rules for `DailyStats` and `AIFeedback`; catch-all narrowed
- [ ] Agreed with the dashboard team
- [ ] Lead: Rules Playground check; existing screens still load
- [ ] Write-up

### P7 - Wrap-up + handoff
- [ ] [[Firestore_Schema]] updated; T-007 marked merged
- [ ] Handoff message finalized and sent by the lead
- [ ] T-011 posted to GitHub Issues
- [ ] Write-up

---

## Mapping to the Boracle platform (later)

Each document maps to one row, so moving to Postgres is a copy, not a redesign:

| Firestore | Postgres (suggested) |
|-----------|----------------------|
| `DailyStats/{date}` | `daily_stats (user_id, date, source, device, timezone, steps, vo2_max_low, vo2_max_high, sleep jsonb, heart jsonb, workouts jsonb, schema_version, synced_at)`, primary key `(user_id, date)` |
| `AIFeedback/{date}` | `ai_feedback (user_id, date, headline, key_signals jsonb, focus jsonb, model, input_tokens, output_tokens, data_source, regenerated_count, created_at)` |

## Out of scope

- `TeamFeedback` (the coach's team assessment): [[T-012_Team_Insights_AI_Feedback_PoC]], designed by the senior design team.
- Day activity totals ([[T-008_Daily_Activity_Stats]]).
- Backfilling past days or weeks.
- Raw per-minute heart-rate documents, sleep-stage documents and GPS routes (see "What is not stored").
- Health Connect as a source ([[T-001_Android_Health_Connect]]); it writes the same document with `source: "health_connect"` after the Fitbit API ends.
- Cleaning up the old collections and test accounts ([[T-013_Firestore_Cleanup_Fresh_Start]]).
- Any Coaching Dashboard change ([[T-011_Dashboard_Athlete_Health_Page]], [[T-012_Team_Insights_AI_Feedback_PoC]]).
- Uploading feedback that existed only on the phone before this ticket (Regenerate re-creates it).
- Moving data to the Boracle platform.
- Fake or seed data.

## Acceptance criteria

- [ ] Opening a finished day on DailySummary creates or overwrites `Users/{uid}/DailyStats/{date}` with every field in the table, `source: "fitbit"`, and nulls (not zeros) for missing values.
- [ ] Each workout entry carries its own duration, calories, distance, pace, steps and average HR, matching the app's workout stats card.
- [ ] `vo2Max` is saved as `{ low, high }` when Fitbit has an estimate and `null` otherwise; a missing scope never breaks the day.
- [ ] `heart.hr5min` has 288 entries for a day the watch was worn; a failed intraday call leaves it `null` without breaking the day.
- [ ] `sleep.stages` matches the hypnogram shown in the app for that night.
- [ ] Today is never saved.
- [ ] After Get feedback, `Users/{uid}/AIFeedback/{date}` has every field; Regenerate overwrites it and increments `regeneratedCount`; `NO_DATA`, errors and cache hits write nothing.
- [ ] Opening the app saves yesterday (and missed days since the last save) without visiting DailySummary; no duplicates.
- [ ] A failed write never breaks DailySummary or the feedback card.
- [ ] The new rules stop one athlete from reading another athlete's `DailyStats` from the client.
- [ ] Unit tests for `toDailyStatsDoc`, the save functions, `hr5min` building, the thunk hooks and the auto-save; `npm test` passes in `SmartClothingApp/` and `insights-service/`.
- [ ] Real days of the lead's data and feedback are accumulating in Firestore **before 2026-10-30**, checked in the Firebase console.

## Estimate

About 2-3 days of work across P1-P6, one short chat each. The riskiest parts are the rules change (P6: the catch-all rule has to be narrowed without locking out existing screens) and getting P1-P5 live before 2026-10-30. T-010's save, T-011 and T-012 all build on it.

## Related Notes

- Phase tickets: [[T-009-P0_Schema_And_Docs]], [[T-009-P1_Service_HR5min_VO2Max]], [[T-009-P2_App_Save_Module]], [[T-009-P3_Save_The_Day]], [[T-009-P4_Save_The_Feedback]], [[T-009-P5_Daily_Auto_Save]], [[T-009-P6_Security_Rules]], [[T-009-P7_Wrap_Up_Handoff]]
- [[Firestore_Health_Schema]]
- [[Firestore_Schema_Proposal]]
- [[T-001_Android_Health_Connect]]
- [[T-013_Firestore_Cleanup_Fresh_Start]]
- [[T-007_Save_AI_Feedback_To_Firestore]] (merged here)
- [[T-008_Daily_Activity_Stats]]
- [[T-010_iOS_Apple_Watch_AI_Feedback_PoC]]
- [[T-011_Dashboard_Athlete_Health_Page]]
- [[T-012_Team_Insights_AI_Feedback_PoC]]
- [[T-003_AI_Feedback_PoC]]
- [[Firestore_Schema]]
- [[Firestore_Rules]]
