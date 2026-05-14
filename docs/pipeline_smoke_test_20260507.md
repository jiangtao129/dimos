# Pipeline Smoke Test — 2026-05-07

Routine re-validation of the `/ship` workflow on `jiangtao129/dimos`,
roughly 10 days after the original bootstrap (see
`docs/pipeline_test_report_20260427.md`). The intent is not to land any
behavioral change but to confirm that the 11-step flow encoded in
`.cursor/commands/ship.md` still produces a green PR end-to-end.

## Scope

- One docs-only file (this one).
- No code, dependency, workflow, or pre-commit hook changes.
- No edits to `scripts/verify.sh`, `conftest.py`, or
  `.github/workflows/verify.yml`.

## What this PR exercises

| Step (per `ship.md`) | Expectation |
|----------------------|-------------|
| 4 — `bash scripts/verify.sh` | `ALL OK` locally (uv sync, pre-commit, pytest fast set) |
| 6 — `git push -u github HEAD` | accepted on the first try, no `--force` |
| 7 — `gh api repos/.../pulls -X POST` | PR opened, no GraphQL race |
| 8 — `gh pr merge --auto --squash --delete-branch` | auto-merge armed |
| 9 — CI `verify` job | passes within the usual 30–90 s warm cache window |
| 10 — Codex auto-review | reaction or `COMMENTED` within ~90 s |
| 11 — squash-merge into `main`, branch deleted | `state=MERGED`, remote branch gone |

## Out of scope

- Branch protection, Codex toggle, repo secrets — all human-owned per
  `AGENTS.md > Pipeline > Ground rules > Rule 6`.
- Any change above the
  `## Pipeline (fork @ jiangtao129/dimos)` anchor in `AGENTS.md`.
- Upstream sync from `dimensionalOS/dimos`.

## Follow-up

If anything in the above table fails, the failing step is the
regression to investigate; the fix lands in a separate, focused PR
(per the "one PR, one focused change" rule).
