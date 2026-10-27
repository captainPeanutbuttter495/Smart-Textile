---
id: T-009-P5
title: "Phase 5: Daily auto-save"
parent: T-009
area: Data
status: Done (skipped-day check open)
assignee: Matthew Garcia
branch: feature/health-schema
pr: —
created: 2026-10-21
updated: 2026-10-26
---

# T-009-P5: Phase 5 - Daily auto-save

Phase 5 of [[T-009_Save_Daily_Stats_To_Firestore]]. It must be live before the Fitbit cutoff on **2026-10-30**.

## Summary

When the app opens or returns to the foreground, it saves yesterday, plus any finished days missed since the last save, without the athlete visiting DailySummary. This gives one `DailyStats` document per day going forward.

## Scope

- On app open or foreground (signed in, watch connected), save yesterday in the background.
- Also save the finished days missed since the last saved date, capped at about 3, so a day without opening the app isn't lost.
- Keep the last saved date in AsyncStorage (`src/utils/localStorage.js`).
- It runs in the background of the UI: it never blocks and never shows an error screen.

## Checklist

- [x] Save on app open / foreground: yesterday plus missed days since the last save (capped at 3)
- [x] Last saved date in AsyncStorage + tests
- [x] `npm test` green in `SmartClothingApp/` (32 suites, 321 tests)
- [x] Lead: doc written on open without opening DailySummary (2026-10-25 deleted, then recreated); no rewrite on foreground
- [ ] Lead: skipped day caught up on the next open
- [x] Live before 2026-10-30 (pushed 2026-10-26)
- [x] Write-up

## Lead checks

- Open the app the next morning: yesterday's document is in the console without visiting DailySummary.
- Skip a day: both days appear on the next open.
- Reopening the app adds no duplicates (the document id is the date).

## Diagrams

### Dynamic - Auto-save on open

```mermaid
%%{init: {'theme': 'base', 'flowchart': {'useMaxWidth': true}, 'themeVariables': {'primaryColor': '#438DD5', 'primaryTextColor': '#fff', 'lineColor': '#666', 'fontSize': '14px'}}}%%
sequenceDiagram
    participant App as App.js (AppState)
    participant AS as AsyncStorage lastSavedDate
    participant S as insights-service
    participant H as healthStore.saveDailyStats
    participant FS as Firestore DailyStats

    rect rgb(40, 85, 140)
        App->>App: opened or back in foreground, signed in, watch connected
        App->>AS: read lastSavedDate
        AS-->>App: 2026-10-23 (today is 2026-10-26)
        App->>App: days to save: 10-24, 10-25 (capped, never today)
    end
    rect rgb(46, 110, 70)
        loop each missed day, oldest first
            App->>S: GET /api/fitbit/day/date
            S-->>App: day
            App->>H: saveDailyStats
            H->>FS: setDoc (id = date, no duplicates)
            App->>AS: lastSavedDate = date
        end
    end
    rect rgb(95, 95, 95)
        Note over App,FS: Any failure stops the loop quietly; the next open tries again from lastSavedDate
    end
```

### Legend

- **Blue block** = deciding which finished days are missing
- **Green block** = loading and saving each one, moving `lastSavedDate` forward after each success
- **Gray block** = failures are silent and retried on the next open

## Files changed

Commits on `feature/health-schema`: `01ca5dc`, `ec185be`, `42ffc90`, `b20d4f3`, `b07d431`.

| File | Change |
|------|--------|
| `SmartClothingApp/src/utils/localStorage.js` | `getLastSavedDate(uid)` / `setLastSavedDate(uid, date)` under the key `lastSavedDate:{uid}`, in the same try/catch style as the feedback cache. Per user, so a second account on the phone starts its own count. A failed read returns `null` |
| `SmartClothingApp/src/actions/deviceActions.js` | `saveFinishedDay` now returns `true` / `false` (P3 behaviour unchanged: errors are still only logged). New `startAutoSaveDays()`: returns if there's no `auth.currentUser`, no Fitbit tokens, or a run is already going (module flag). It saves from the day after the last saved date (no earlier than 3 days ago) through yesterday, oldest first: `getFitbitDay`, `fitbitDayLoaded` (so opening that day later is instant), `saveFinishedDay`, then `setLastSavedDate`. A failed fetch goes through `startHandleFitbitError` and stops; a failed write stops. Nothing is shown to the user |
| `SmartClothingApp/App.js` | Dispatches `startAutoSaveDays()` after `loadAppResources` finishes (auth is restored by then), and again from an `AppState` listener whenever the app becomes `active` |
| `__test__/__unit-test__/__actions-test__/deviceActions.unit.test.js` | Mocks `localStorage`; 14 tests under `startAutoSaveDays` with the clock fixed at 2026-10-11 |
| `__test__/__unit-test__/__utils-test__/localStorage.unit.test.js` | 5 tests under "last saved date" |

The DailySummary save from P3 does **not** move the last saved date, otherwise opening yesterday by hand would skip an earlier missed day.

## Tests

`npm test` in `SmartClothingApp/`: **32 suites, 321 tests, all passing** (302 before P5). The new tests:
- `startAutoSaveDays`: saves yesterday and moves the last saved date; puts each saved day in the store; catches up missed days oldest first; goes back at most 3 days; saves the last 3 days on the first run; does nothing when yesterday is already saved; never saves today; does nothing without a signed-in user; does nothing without a Fitbit connection; stops at a day that fails to load and keeps the last saved date; disconnects when Fitbit needs a new login; stops at a failed write; ignores a second run while one is in progress; runs again after the previous run finishes
- last saved date: stored under a key for the user; read for the user; `null` when nothing was saved; `null` when reading fails; only logs when storing fails

## Manual verification

Done by the lead on 2026-10-26 on the phone, with the service on `npm start`:
- **Save on open:** deleted `Users/{uid}/DailyStats/2026-10-25` in the console, opened the app (it landed on Home, DailySummary never opened), and the 2026-10-25 document was written again while the console was open.
- **No rewrite on foreground:** switched to Discord and back; `syncedAt` on 2026-10-25 didn't change, since yesterday was already the last saved date.
- **Still open:** skip opening the app for a day, then check that both missed days appear on the next open. The lead will do this before 2026-10-30.

## Deviations from the plan

- **The first run goes back 3 days, not just to yesterday.** With no last saved date, the auto-save re-saves the last 3 finished days. Days P3 already saved are overwritten with the same data (one extra write each, once).
- **The cap is 3 days.** A phone that isn't opened for longer loses the older days unless they're opened on DailySummary, which still saves any finished day.
- **The auto-save is started from `App.js`, not a screen,** so it runs whatever screen the app opens on.

## Follow-ups

- Skipped-day check (lead, before 2026-10-30).
- After the Fitbit cutoff the auto-save has nothing to fetch until [[T-001_Android_Health_Connect]] builds the day on the phone; `startAutoSaveDays` will need that source instead of `getFitbitDay`.
- AI feedback is not auto-generated; it stays a paid, user-triggered call (see T-016).

## Related Notes

- [[T-009_Save_Daily_Stats_To_Firestore]]
- [[T-009-P3_Save_The_Day]]
- [[T-001_Android_Health_Connect]]
- [[Firestore_Health_Schema]]
