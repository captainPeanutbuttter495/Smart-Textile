# Tickets

Every ticket handed to the team is drafted here first, then posted to GitHub Issues. This folder is the record of what has been written, who has it, and where it stands.

---

## Workflow

1. Copy `_Template.md` to `T-NNN_Short_Title.md` (next number from the table below).
2. Fill in every section; keep acceptance criteria testable.
3. Post the body (everything below the frontmatter) as a GitHub issue in `CSU-Northridge-ARCS-Dev/smart-clothing-app`.
4. Paste the issue URL into the ticket's `github_issue` field and into the table below; set status to **Posted**.
5. Update `status` / `assignee` as the ticket moves. When the PR merges, set **Done** and link the PR in the ticket.

## Status legend

| Status | Meaning |
|--------|---------|
| Draft | Written here, not yet on GitHub |
| Posted | GitHub issue exists, unassigned |
| In Progress | Assigned and being worked on a feature branch |
| In Review | PR open against `main` |
| Done | Merged to `main` |
| Dropped | Closed without merging (say why in the ticket) |

---

## Index

| ID | Title | Area | Priority | Status | Assignee | GitHub Issue | Created | Doc |
|----|-------|------|----------|--------|----------|--------------|---------|-----|
| T-001 | Android: connect a smartwatch via Health Connect (heart rate, sleep, steps) | Android / Data | High | Draft | — | — | 2026-09-13 | [[T-001_Android_Health_Connect]] |
| T-002 | iOS: connect an Apple Watch via HealthKit (heart rate, sleep, steps) — depends on T-001; documents the state of `ios-dev` | iOS / Data | Medium | Draft | — | — | 2026-09-14 | [[T-002_iOS_Apple_HealthKit]] |

---

## Backlog ideas (not yet ticketed)

Candidates surfaced while documenting `main`; promote to a ticket when ready.

| Idea | Source doc |
|------|-----------|
| Fix `HeartRateChart` prop mismatch on Home (`data` vs `dataArray`) | [[Home_Dashboard]] |
| Hide or remove Breath Rate / Ventilation charts until a Hexoskin data path exists | [[Home_Dashboard]] |
| Tighten Firestore rules (per-user access, coach role checks) with the dashboard team | [[Firestore_Rules]] |
| Reconcile `disabledCoachList` vs `CoachListDisabled` and wire the Stop/Share toggle | [[Coach_Access_Permissions]] |
| `updateUserEmail` should also update `Users/{uid}.email` (Cloud Function matches on it) | [[Firebase_Auth]] |
| Implement or remove the Settings "DELETE DATA" button | [[Profile_Settings]] |
| Remove dead navigators, `src/network/`, unused deps, `GET_DEVICES` reducer case | [[Navigation]], [[Devices_Tab]] |
| CI: drop `-u`, add lint, run on pushes to `main` / `stable` | [[CI_GitHub_Actions]] |
| HRV, resting HR, activity rings from HealthKit (follow-up to T-002; `origin/ios-dev` read them) | [[T-002_iOS_Apple_HealthKit]] |
| Fitbit Web API via Cloud Functions as a second `source` for background sync | [[T-001_Android_Health_Connect]] |
