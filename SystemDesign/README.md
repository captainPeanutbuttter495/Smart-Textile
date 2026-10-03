# System Design

Architecture documentation for the Smart Clothing App: C4 model diagrams and the design decisions they make visible.

---

## C4 Diagrams

The `C4-Diagrams/` subfolder follows the [C4 model](https://c4model.com/) (Context, Containers, Components, Code) by Simon Brown.

| Level | Note | Description |
|-------|------|-------------|
| — | [[00-C4-Overview]] | Index, diagram palette and init line, key architecture decisions |
| **0** | [[01-SystemLandscape]] | Athletes, coaches, the mobile app, the Coaching Dashboard, Firebase, Expo Push, planned wearables |
| **1** | [[02-SystemContext]] | The mobile app and its direct relationships with users and external systems |
| **2** | [[03-Container]] | React Native app, Cloud Functions, AsyncStorage; Firebase Auth, Firestore, Expo Push |
| **3a** | [[04-Component-Frontend]] | Boot sequence, navigation, screens, Redux, actions, utils |
| **3b** | [[05-Component-Backend]] | Firestore collections, security rules, the invitation Cloud Function |
| **4** | [[06-Code]] | Firestore schema, Redux state, boot / auth state machine, pending-permissions check |
| **Dyn** | [[07-Dynamic-Diagrams]] | Login / restore, sign-up, coach invite -> push -> permissions, health query, logout |

All diagrams use **Mermaid**, which renders natively in Obsidian and on GitHub. The palette and rules are in [[00-C4-Overview]]; the full conventions are in the `docs-sync` skill (`SmartClothingApp/.claude/skills/docs-sync/SKILL.md`).

---

## Related Notes

- [[Tech_Stack]]
- [[Project_Structure]]
- [[Firestore_Schema]]
- [[Roadmap]]
