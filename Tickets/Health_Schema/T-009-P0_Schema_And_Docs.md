---
id: T-009-P0
title: "Phase 0: Schema and docs"
parent: T-009
area: Docs
status: Done
assignee: Matthew Garcia
branch: — (docs only)
pr: —
created: 2026-10-21
updated: 2026-10-21
---

# T-009-P0: Phase 0 - Schema and docs

Phase 0 of [[T-009_Save_Daily_Stats_To_Firestore]]. Docs only, no code.

## Summary

The Firestore contract for each athlete's day (`DailyStats/{date}`) and that day's Claude feedback (`AIFeedback/{date}`) was reviewed field by field with the lead. Then it was cut down to two collections and split into phases P0-P7. Every later phase builds against this contract.

## Scope

- A field-by-field review with the lead. The decisions are in the master ticket's "Decisions" table:
  - no `activity` field
  - no `promptVersion`
  - `TeamFeedback` moves to T-012
  - no "Upload last 7 days" button
  - `hr5min` at 5-minute resolution
  - a new `vo2Max` field
- Rewrite the master ticket with phases and a checklist, and move it to `Tickets/Health_Schema/`.
- Create [[Firestore_Health_Schema]] (diagrams and field reference), and update [[Firestore_Schema_Proposal]].
- Draft the senior design handoff (not sent; it waits for real data after P3).

## Checklist

- [x] Field-by-field review; `activity`, `promptVersion`, `TeamFeedback` and the 7-day upload cut
- [x] Master ticket rewritten with phases and checklist, moved to `Tickets/Health_Schema/`
- [x] [[Firestore_Health_Schema]] created; [[Firestore_Schema_Proposal]] updated
- [x] Senior design handoff drafted (not sent)
- [x] `vo2Max` added to the schema (after the 2026-10-21 call with Carlos)
- [x] One ticket per phase (T-009-P0 to P7) created in `Health_Schema/`

## Diagrams

The overview, one `DailyStats` document, one `AIFeedback` document and "when the app writes" are all in [[Firestore_Health_Schema]]. They aren't repeated here.

## Files changed

| File                                                           | Change                                          |
| -------------------------------------------------------------- | ----------------------------------------------- |
| `Tickets/Health_Schema/T-009_Save_Daily_Stats_To_Firestore.md` | rewritten: decisions, schema, phases, checklist |
| `Backend/Firestore_Health_Schema.md`                           | added: diagrams and field reference             |
| `Backend/Firestore_Schema_Proposal.md`                         | status box with the changes                     |
| `Tickets/Senior_Design/Handoff_Message_DRAFT.md`               | drafted                                         |
| `Tickets/Health_Schema/T-009-P0..P7_*.md`                      | added: one ticket per phase                     |

## Deviations from the plan

- The phase tickets were created up front, not as write-ups afterward (lead, 2026-10-21). Each one is filled in when its phase ends.
- The master ticket stays the checklist.
- Work happens on `feature/health-schema`, branched from `feature/insights-service`. That branch counts as the main branch for now; nothing merges to `main`.

## Related Notes

- [[T-009_Save_Daily_Stats_To_Firestore]]
- [[T-009-P1_Service_HR5min_VO2Max]]
- [[Firestore_Health_Schema]]
- [[Firestore_Schema_Proposal]]
