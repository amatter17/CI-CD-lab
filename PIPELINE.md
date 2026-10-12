# Pipeline Design

## 1. Triggers

| Event | Why |
|---|---|
| `push` to `main` (ignoring `*.md` and `docs/**`) | Verify that merged code is healthy. Docs-only changes cannot break behavior, so running CI for them wastes minutes. |
| `pull_request` targeting `main` | Catch problems before merge. This is the core CI trigger. |
| `workflow_dispatch` (inputs: `python-version`, `create-package`) | Manual runs for debugging and for testing another Python version. |
| `schedule` (Mondays 07:00 UTC) | Detect breakage caused by external changes (new dependency releases) even when no code changed. |

## 2. Jobs and Job Graph

```text
test ──► package ──► verify
  │          │
  │          └────► pr-comment   (pull requests only)
  │
  └──────────────┐
                 ▼
   package ──► report   (needs: test, package; runs always)
```

- `test` runs first, with no dependencies.
- `package` waits for `test`.
- `verify` waits for `package`.
- `pr-comment` waits for `package` and only runs on pull requests.
- `report` waits for `test` and `package` and runs with `if: always()`, in parallel with `verify`.

## 3. Artifacts

- `package`: `temperature-<version>.tar.gz` (contains `temperature.py` and `README.md`). This is the deliverable.
- `test-report`: `junit.xml`. This is evidence, uploaded even when tests fail.

## 4. CI, CD, or Continuous Deployment?

**Continuous Integration only**, because the pipeline builds, tests, and verifies a package but never releases or deploys it anywhere.

## 5. When tests fail

The `package` job is **skipped**. Building a package from broken code is pointless and risky, because someone could use it by mistake. This is the fail-fast principle.
