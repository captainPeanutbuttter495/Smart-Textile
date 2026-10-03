# CI: GitHub Actions

The single workflow in `.github/workflows/main.yml` at the git root.

---

## Overview

| Setting | Value |
|---------|-------|
| Name | "Test React Native App testing" |
| Triggers | `pull_request` (any branch), `push` to `testing`, `workflow_dispatch` |
| Runner | `ubuntu-latest`, Node `18.x` |
| Steps | `actions/checkout@v2`, `actions/setup-node@v1`, `npm install`, `npm test -- -u` (both with `working-directory: SmartClothingApp`), echo success |

---

## Consequences

- **Snapshot drift never fails CI.** `-u` rewrites snapshots on the runner, so only assertion failures and crashes are caught.
- **No lint step.** `eslint.config.js` in the app is not executed; `functions/` lint only runs at deploy time.
- **No build step.** Native compilation and Expo config are not validated.
- **Pushes to `main` / `stable` do not run tests** (only PRs targeting them do), so direct pushes are unchecked.
- Uses deprecated action majors (`checkout@v2`, `setup-node@v1`); GitHub prints warnings but they still run.

---

## Branching model it supports

| Branch | Role |
|--------|------|
| `stable` | Production-ready |
| `main` | In-progress but must stay functional; feature branches merge here via reviewed PRs |
| `testing` | Push target that also triggers the workflow |
| feature branches | Open a PR to `main` |

---

## Suggested improvements (not implemented)

- Drop `-u` so snapshot changes must be committed deliberately.
- Add `npx eslint src` once `eslint.config.js` is cleaned up.
- Add `push: branches: [main, stable]` to the triggers.

---

## Related Notes

- [[Testing]]
- [[Project_Structure]]
