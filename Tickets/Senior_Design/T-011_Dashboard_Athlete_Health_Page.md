---
id: T-011
title: "Coaching Dashboard: an athlete's Health page with their own sleep, heart rate, workouts and saved AI feedback from Firebase"
area: Coaching Dashboard / UI
priority: Medium
status: Draft
assignee: Senior design team
github_issue: —
created: 2026-10-17
updated: 2026-10-17
---

# T-011: Coaching Dashboard: an athlete's Health page with their own sleep, heart rate, workouts and saved AI feedback from Firebase

## User story

> A user with the **athlete** role signs in to the Coaching Dashboard and is sent to `/athlete`. Today that page says "This is the Athlete's Dashboard", and the navbar's **Heart** and **Sleep** links lead nowhere. After this ticket, the athlete lands on a **Health** page. They pick a day and see their own sleep, heart rate, daily stats and workouts, plus the AI feedback the Smart Clothing App saved for that day. A month view shows the days they worked out.

## Summary

This is the dashboard mirror of the app's DailySummary screen ([[T-003_AI_Feedback_PoC]] P2/P4). It is a **read-only** page: everything comes from Firebase, written by the Smart Clothing App. It makes no Claude calls; the feedback is the one the athlete already generated in the app.

## Why

- Dr. Jiang (2026-10-13): the dashboard needs real data, not hard-coded data. The senior design team mirrors the AI feature on the dashboard.
- The athlete side of the dashboard isn't built: `/athlete` is a placeholder, and `/heart` and `/sleep` are not routes in `src/App.jsx` (`coaching-dashboard` @ `7cde460`).
- It was decision 2 in [[T-003-P6_Presentation_Walkthrough]]: one Health page instead of separate heart and sleep pages. The lead chose one page on 2026-10-17.

## Depends on

- [[T-009_Save_Daily_Stats_To_Firestore]]: the shared Firestore health schema, `Users/uid/DailyStats/date` (with `sleep.stages` and `heart.hr5min`) and `Users/uid/AIFeedback/date` (T-007 is merged into it).
- Real data only: no fake or seed data in Firebase. Real days start appearing once T-009 P3 ships and accumulate daily from then on (no backfill). The API and page can be built and tested against the schema with fakes in tests before that.
- Schema diagrams and field reference: [[Firestore_Health_Schema]].

## Scope

1. **API** (`server/server.js`, new routes). Each route verifies the Firebase ID token and uses **the caller's own uid**; an athlete can never pass another `userId`.

   | Method | Path | Returns |
   |--------|------|---------|
   | GET | `/api/athlete/day?date=YYYY-MM-DD` | The `DailyStats` document for that day (or `null`; it includes the sleep stages and the 5-minute heart-rate timeline) and the `AIFeedback` document (or `null`). Two reads |
   | GET | `/api/athlete/month-workouts?month=YYYY-MM` | `[ date, workouts ]` for the days that have workouts |

2. **Page** `src/pages/AthleteHealth.jsx` at `/athlete` (replacing the placeholder). It has a date picker (default yesterday, using `PopupCalendar`) and these cards:
   - **Sleep:** total, efficiency, deep / light / REM / awake minutes, and the stage timeline from `sleep.stages` with `SleepTimeline`.
   - **Heart:** resting HR, the change from the day before, cardio and peak zone minutes, and a day-long heart-rate line from `heart.hr5min` (288 five-minute averages).
   - **Daily stats:** steps against 10,000. (T-009 has no day activity totals; per-workout energy burned, distance and pace are on the Workouts card.)
   - **Workouts:** for each session, type, duration, energy burned (`calories`), steps, avg HR, distance and pace (miles, like the app), from that `workouts[]` entry.
   - **AI feedback:** headline, key signals and focus, read-only, with the date it was generated. If there is none: "No AI feedback for this day. Generate it in the Smart Clothing App."
   - **Workout calendar:** the month with workout days marked; tapping a day selects it.
3. **Navbar:** replace the athlete's Heart and Sleep links with one **Health** link.
4. **Sleep stages:** T-009 stores stages as names (`deep`, `light`, `rem`, `wake`; classic nights `asleep`, `restless`, `awake`) inside `DailyStats`, not as `SleepDataHC` documents. Add a small adapter that turns `sleep.stages` into the points `SleepTimeline` draws (its own codes are 1 Awake, 2 REM, 3 Light, 4 Deep). Don't change how `SleepTimeline` reads `SleepDataHC`, so the coach pages that use it still render.
5. Use `apiUrl()` and show errors instead of empty cards (dashboard backlog item).

## Out of scope

- Generating AI feedback on the dashboard (it's read-only here; team feedback is [[T-012_Team_Insights_AI_Feedback_PoC]]).
- The coach's `/info` page and the coach home page.
- Per-minute heart-rate charts (T-009 stores 5-minute averages only).
- Route maps (GPS isn't uploaded).
- Full API auth (SEC-1); only the new routes check tokens.

## Acceptance criteria

- [ ] An athlete account signing in lands on the Health page; picking a day with data shows sleep, heart, daily stats, workouts and the saved AI feedback for that day.
- [ ] A day without data shows clear empty states, not zeros or a blank page.
- [ ] The workout calendar marks the days with workouts in the selected month.
- [ ] The routes return 401 without a valid token, and only ever return the caller's own data.
- [ ] Sleep stages from `sleep.stages` show the right labels, and the coach pages that read `SleepDataHC` still render.
- [ ] The athlete navbar has one Health link and no dead links.
- [ ] `npm run lint`, `npm run build` and `npm run test:run` pass, with tests for the new routes and the page states.

## Estimate

About 1 week for 1-2 students. The riskiest step is the sleep-stage mapping without breaking the coach pages.

## Related Notes

- [[Tickets/README|Tickets]]
- [[T-012_Team_Insights_AI_Feedback_PoC]]
- [[T-009_Save_Daily_Stats_To_Firestore]]
- [[T-003_AI_Feedback_PoC]]
