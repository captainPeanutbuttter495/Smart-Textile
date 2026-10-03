# Smart Clothing App

A React Native + Expo mobile app for CSUN student-athletes, built by the ARCS program's senior-design teams. Athletes create an account, enter body metrics, and grant coaches access to their data; coaches review that data in the separate Coaching Dashboard. The long-term goal is to ingest vital signs from smart textiles and smartwatches; on `main` today no wearable integration exists and the health charts are placeholders.

This vault documents the app **as it is on `main`**, not as planned. Hard-coded data is labelled hard-coded. The vault lives at `C:\Users\mbg91918\Smart-Clothing-Docs`, outside the repo. Open that folder as a vault in [Obsidian](https://obsidian.md) to get rendered diagrams and working links.

---

## Current State (as of 2026-10-02, `main @ f65e19f`)

- **Working:** email / password auth, profile and body metrics, coach invites with Expo push, the pending-permission and coach-access modals.
- **Placeholders:** Home cards and charts, Insights rings, the Devices list, and parts of the Health screens are **hard-coded**. See [[Home_Dashboard]].
- **No wearable data path exists.** The health screens query collections nothing writes. Next up: [[T-001_Android_Health_Connect]], then [[T-002_iOS_Apple_HealthKit]]. See [[Roadmap]].
- **Security:** Firestore rules allow any signed-in user to read and write everything. See [[Known_Issues]] #1.
- **CI** runs `npm test -- -u`, so snapshot drift never fails a build. See [[CI_GitHub_Actions]].

---

## Tech Stack

### Mobile app — `SmartClothingApp/`
| Technology | Purpose |
|-----------|---------|
| React Native 0.76 + Expo SDK 52 | Cross-platform framework (dev client; Expo Go for non-push work) |
| React Navigation 6 | Auth/main switch, bottom tabs, native stacks |
| Redux 4 + redux-thunk | Global state; all Firebase calls in thunks |
| react-native-paper (MD3) | UI components + theme |
| Firebase JS SDK v10 | Auth + Firestore from the client |
| AsyncStorage | Session cache + Firebase Auth persistence |
| expo-notifications | Push token registration + tap handling |
| react-native-chart-kit, victory-native + Skia | Charts and activity rings |

### Backend — Firebase
| Technology | Purpose |
|-----------|---------|
| Firebase Authentication | Email/password identity |
| Cloud Firestore | System of record, shared with the Coaching Dashboard |
| Cloud Functions (`functions/`) | `sendInvitationNotification` push trigger |

### External Integrations
| Service | Purpose |
|---------|---------|
| Expo Push Service | Delivers coach-invite notifications |
| Coaching Dashboard (separate repo) | Writes invitations, reads athlete data |
| Google Health Connect / Apple HealthKit / Hexoskin | **Not implemented** (see Tickets) |

### Testing
| Layer | Tools |
|-------|-------|
| App | Jest (`jest-expo`), React Native Testing Library, manual mocks |
| Functions | none |

---

## Repository Structure

```
smart-clothing-app/                  git root
├── .github/workflows/main.yml       CI: npm test -- -u
└── SmartClothingApp/                the Expo app (all npm commands run here)
    ├── App.js, firebaseConfig.js, app.json, .env
    ├── src/{actions,reducers,navigation,screens,components,utils,constants}
    ├── functions/                   Cloud Functions package
    ├── __test__/                    Jest suites + mocks
    ├── .claude/skills/docs-sync/    vault + diagram conventions
    ├── .claude/commands/            /session-log
    └── CLAUDE.md
```

---

## Getting Started

```bash
cd SmartClothingApp
npm install
# get .env and google-services.json from an admin (see Environment_Setup)
npx expo start --offline --localhost     # Expo Go on the emulator (no push)
npm run android                          # Android dev build (push works)
npm test
```

---

## SDLC Process

1. Ticket drafted in `Tickets/` from `_Template.md`, then posted to GitHub Issues
2. Feature branch off `main`
3. Tests added alongside the change (see [[Testing]])
4. PR to `main` (CI runs `npm test -- -u`); reviewed and merged
5. `main` promoted to `stable` when release-ready
6. Docs in this vault updated in the same PR cycle (`/docs-sync` conventions)

---

## Documentation

### Quick Links

- [[Tech_Stack]] — Technologies and versions
- [[Project_Structure]] — Repository file organization
- [[Testing]] — Test strategy, tooling, and suites
- [[Environment_Setup]] — `.env`, native config files, running the app
- [[Known_Issues]] — verified defects and risks on `main`
- [[Roadmap]] — built vs. planned

---

### Frontend

- [[Authentication]] — Sign in / up / reset, session restore, logout
- [[Navigation]] — AuthStack vs AllNavigationStack, tabs, routes
- [[Redux_State]] — Store, reducers, action conventions
- [[Home_Dashboard]] — Home tab: what is real vs hard-coded
- [[Health_Data_Screens]] — Heart rate + sleep screens, date toolbar, Firestore queries
- [[Insights_Activity_Rings]] — Insights screen and Skia rings (mock data)
- [[Devices_Tab]] — Static device list; no connectivity on `main`
- [[Profile_Settings]] — Profile, metrics modal, Settings modals
- [[Coach_Access_Permissions]] — Pending invites, coach list, permission modals
- [[Push_Notifications_Client]] — Token registration and notification handling

---

### Backend

- [[Firebase_Auth]] — Auth operations and session model
- [[Firestore_Schema]] — Every collection and document shape (app + dashboard)
- [[Firestore_Rules]] — Current rules and the security gap
- [[Cloud_Functions]] — `sendInvitationNotification`

---

### Infrastructure

- [[Firebase_Setup]] — Client init, console settings, deploying rules/functions
- [[Android_Emulator_Workflow]] — Known-good emulator flags and failure modes
- [[iOS_EAS_Build]] — EAS dev client for iOS
- [[CI_GitHub_Actions]] — The workflow and its blind spots

---

### System Design

- [[00-C4-Overview]] — C4 index, diagram palette, key decisions
- [[01-SystemLandscape]] · [[02-SystemContext]] · [[03-Container]]
- [[04-Component-Frontend]] · [[05-Component-Backend]] · [[06-Code]] · [[07-Dynamic-Diagrams]]

---

### Sessions

A dated log of work sessions, written by `/session-log`. See [[Sessions/README|Session index]].

---

### Tickets

- [[Tickets/README]] — Ticket index, workflow, status legend, backlog ideas
- [[T-001_Android_Health_Connect]] — First wearable ticket: Health Connect heart rate / sleep / steps

---

## Conventions

- File names use `PascalCase_Underscores` and **must be unique across the vault**, because wiki-links resolve by name. Folder indexes are `README.md` and are linked with their path (`[[Tickets/README|Tickets]]`). C4 files are numbered `NN-Name.md`; tickets are `T-NNN_Short_Title.md`.
- Content notes have no frontmatter and follow one layout: `# Title`, a one-line summary, `---`-separated sections, **Key Files**, **Tests**, then **Related Notes**. Ticket and Session notes carry frontmatter from their template.
- Diagrams are **Mermaid** (native in Obsidian). The init line and palette are in [[00-C4-Overview]]. Put a `### Legend` after every diagram.
- **Accuracy rule:** document what is on `main`. Anything planned but not built is labelled *(planned)* and drawn gray and dashed. Record only real numbers; never invent metrics.
- **Domain notes say what is true now; session notes say what happened on a day.** If a change makes a domain note wrong, fix the domain note in the same PR.
- **What does not belong here:** secrets, `.env` values, `google-services.json`, real athlete data, or file listings git already records.
- Claude keeps this vault in sync with the `docs-sync` skill (`SmartClothingApp/.claude/skills/docs-sync/`) and the `/session-log` command (`SmartClothingApp/.claude/commands/`).
