# Pipeline Test Report — 2026-04-27

End-to-end validation of the agent-driven dev pipeline bootstrapped on
`jiangtao129/dimos` per `docs/dimos_pipeline_setup_for_agent.md`. This
report records 6 pull requests, 9 CI runs, and 2 distinct Codex review
outcomes that together demonstrate the pipeline is operational.

## Decision recap

| Decision | Choice |
|----------|--------|
| Q1 remote | A — keep `origin`=GitLab, add `github` for jiangtao129/dimos |
| Q2 baseline | B — clean current HEAD; the setup-guide `.md` lands as part of bootstrap PR, not as a separate PR |
| Q3 verify | B — medium: `uv sync` + `pre-commit run --all-files` + `pytest` |
| Q4 LFS push | B — pointer-only (no LFS objects on jiangtao129/dimos) |
| Q5 protection | B — medium: PR + CI green required, no approval required |
| D1 main branch | A — push `feat/jiangtao` directly to `github main` |
| D2 GitHub repo | B — `--force-with-lease` overwrite the stale upstream-fork snapshot |
| D3 upstream workflows | B — gate every job with `if: github.repository == 'dimensionalOS/dimos'`; add new `verify.yml` |
| D7 PR template | C — keep `pull_request_template.md` as-is, `.cursor/commands/ship.md` owns the checklist |

## PR timeline

| PR | Title | Created | Closed/Merged | mergeStateStatus | verify CI | Codex |
|----|-------|---------|---------------|------------------|-----------|-------|
| [#1](https://github.com/jiangtao129/dimos/pull/1) | bootstrap fork pipeline | 02:51:33Z | MERGED 02:53:28Z (immediate, bootstrap exception) | n/a | failure 53s (no portaudio) | none (closed before bot reacted, see Pitfall #5) |
| [#2](https://github.com/jiangtao129/dimos/pull/2) | install portaudio + turbojpeg in verify | 03:09:52Z | MERGED 03:10:20Z (immediate, bootstrap exception) | n/a | success 11m23s | none (closed before bot reacted) |
| [#3](https://github.com/jiangtao129/dimos/pull/3) | gitignore extra patterns | 05:40:53Z | MERGED 05:51:43Z (after CI) | **BLOCKED → CLEAN** | success 10m49s | **`+1` reaction** |
| [#4](https://github.com/jiangtao129/dimos/pull/4) | (intentional) tolerate flaky pytest `\|\| true` | 05:54:38Z | CLOSED 05:57:47Z (manual, P0 demo) | n/a (no auto-merge) | success 12m24s* | **`COMMENTED` review with P0 inline** |
| [#5](https://github.com/jiangtao129/dimos/pull/5) | remove paths-ignore from verify | 06:02:50Z | MERGED 06:17:11Z (after CI) | **BLOCKED → CLEAN** | success 13m57s | `+1` reaction |
| [#6](https://github.com/jiangtao129/dimos/pull/6) | this report | (this PR) | (auto-merge enabled) | (in flight) | (in flight) | (in flight) |

*PR #4 is the cautionary tale: with `|| true` appended, `pytest` itself
failed yet `verify.sh` exited 0 and the `verify` job reported success.
Codex's review explicitly called out exactly this failure mode. If the
PR had been opened with `--auto`, branch protection alone would NOT
have caught the bypass — it relies on the review being read.

## CI run roster

| run_id | branch | event | conclusion | duration |
|--------|--------|-------|------------|----------|
| 24974279393 | chore/agent-pipeline-bootstrap | pull_request | failure | 53s |
| 24974284039 | main (post #1) | push | failure | 54s |
| 24974685275 | fix/verify-system-deps | pull_request | success | 11m34s |
| 24974689087 | main (post #2) | push | success | 11m23s |
| 24978504400 | chore/gitignore-extra-patterns | pull_request | success | 10m49s |
| 24978816386 | main (post #3) | push | success | 10m58s |
| 24978914974 | test/codex-p0-verify-bypass | pull_request | success* | 12m24s |
| 24979161658 | fix/verify-trigger-on-md | pull_request | success | 13m57s |
| 24979596719 | main (post #5) | push | (started 06:17Z) | — |

The fork-gated upstream workflows (`ci`, `code-cleanup`, `macos`,
`docker-build`, `stale`) all logged `skipped` 0–2 s for every PR, as
designed (`if: github.repository == 'dimensionalOS/dimos'`).

## Codex behavior

### PR #3 — clean change, terminal `+1`

```
chatgpt-codex-connector[bot]: 👍 reaction on the PR description
```

Per setup-guide Pitfall #6, `+1` is the terminal "I looked, no issues"
state. No formal review left. PR description's reactions row in the
GitHub UI is the only place this signal is visible.

### PR #4 — deliberate P0 violation, formal `COMMENTED` review

Codex left a formal `COMMENTED` review and an inline comment on
`scripts/verify.sh:74` (the line that gained `|| true`). Full body:

> **P0 Badge — Preserve pytest exit status in verify gate**
>
> Appending `|| true` to the `uv run pytest -q --maxfail=5` command
> suppresses all test failures, so `scripts/verify.sh` reports success
> even when the test suite is red. Because the `verify` CI job runs
> this script directly (`.github/workflows/verify.yml`, step "Run
> unified verify"), this change weakens the repository's required gate
> and can allow regressions to merge unnoticed.

Six things Codex got right, in order:

1. Severity classification: P0 — matched the entry in
   `AGENTS.md > Codex review guidelines > P0` for "Edits to
   `scripts/verify.sh` ... that weaken the verify gate".
2. Title: "Preserve pytest exit status in verify gate" — names the
   fix, not just the symptom.
3. Cause: `|| true` suppresses test failures.
4. Effect: `verify.sh` returns 0 even with red pytest.
5. Cross-reference: cites `verify.yml` step "Run unified verify".
6. Impact: regressions can land unnoticed.

This validates that **Codex actually reads our `AGENTS.md` review
guidelines section and applies the P0/P1/P2/P3 categories**, rather
than just running a generic prompt.

## Three places Codex feedback shows up in the GitHub UI

When inspecting any PR for Codex feedback, check in this order:

1. **PR description's reactions row** (bottom-right of the description
   box). Shows `+1`, `eyes`, `heart`. PR #3 has `+1` here.
2. **Conversation timeline → "Codex Review" collapsible block** plus
   per-line inline comments on the changed-files tab. Only present when
   Codex left a formal review with body. PR #4 has both.
3. **Reviewers list (right sidebar of the PR)**. The
   `chatgpt-codex-connector[bot]` shows up here once a `COMMENTED`
   review is filed; absent when only a reaction was left.

If none of the three carry any Codex signal **after waiting 90 s**,
suspect Pitfall #5: the PR closed too fast (e.g. immediate merge with
no branch protection) and Codex didn't get a chance.

## main history (after PR #5)

```
a4a46362d fix(ci): remove paths-ignore from verify so doc-only PRs are not deadlocked (#5)
3e94293e9 chore(gitignore): add commonly missed editor swap and tool cache patterns (#3)
0b899ce86 fix(ci): install portaudio + turbojpeg in verify workflow (#2)
dcfa6ebaa chore: bootstrap fork pipeline for jiangtao129/dimos (#1)
0611ab41f ci(macos): add macos_bug marker and skip known-crashing worker tests (#1786)  ← baseline (upstream dev tip)
```

## Definition of Done verification (setup-guide §7)

| Item | Status | Evidence |
|------|--------|----------|
| `git remote -v` shows origin (gitlab) + upstream + github | ✅ | three remotes listed |
| `bash scripts/verify.sh` ALL OK locally | ✅ | 50 s on a clean dev machine; 1378 passed, 116 skipped, 352 deselected |
| `verify.yml` calls `bash scripts/verify.sh` as terminal step | ✅ | step "Run unified verify" |
| `AGENTS.md` has appended Pipeline + Codex guidelines section | ✅ | line 395+, no edits above |
| `.cursor/rules/00-workflow.mdc` + `.cursor/commands/ship.md` + `.github/PULL_REQUEST_TEMPLATE.md` in place | ✅ | all three present |
| Bootstrap PR merged | ✅ | PR #1 + PR #2 |
| At least 1 real demo PR through the full flow | ✅ | PR #3 ran the full 11-min branch-protection-gated cycle |
| Branch protection: `contexts: ['verify']`, `enforce_admins: true` | ✅ | verified via `gh api repos/.../branches/main/protection` |
| Codex auto-review on, demo PR sees bot signal | ✅ | PR #3 `+1`, PR #4 P0 inline |
| `docs/pipeline_test_report_<date>.md` archived | ✅ | this file (PR #6) |

## Pitfalls actually hit (vs. the setup guide §5)

### Newly discovered (not in the setup guide)

`paths-ignore: '**.md'` on `verify.yml` triggers conflicts with branch
protection. With `contexts: ['verify']` and `enforce_admins: true`, a
doc-only PR creates no `verify` check and deadlocks at
`mergeStateStatus: BLOCKED` with no recovery path. Fix: remove the
filter (PR #5). Trade-off: doc-only PRs now pay ~11 min CI cost.

### Setup-guide pitfalls confirmed empirically

- **#3** "branch protection 勾了 checkbox 但没把具体 check 加到
  contexts" — pre-empted by the explicit
  `gh api .../branches/main/protection` check in the bootstrap TODO list
  (stage 11). Verified: `contexts: ['verify']`, not `[]`.
- **#5** "PR 太快被 merge,Codex 来不及 review" — observed on PR #1
  and PR #2 (bootstrap exception, no protection yet). PR #3 onward
  proves branch protection's required-CI-green wait gives Codex enough
  time.
- **#6** Codex reaction semantics: `+1` is terminal "looked, no
  issues"; a `COMMENTED` review with body is real findings. Confirmed
  on PR #3 (`+1`) vs PR #4 (`COMMENTED` + P0 inline).

### Setup-guide pitfalls NOT hit on this fork

- #1 SSH connection refused — already configured during
  `unitree_sdk_jt` setup; `~/.ssh/config` reused unmodified.
- #2 GraphQL race on `gh pr create` — pre-empted by always using
  `gh api repos/.../pulls -X POST` (REST), encoded in
  `.cursor/commands/ship.md` step 7.
- #4 `enforce_admins: false` — covered by ticking "Do not allow
  bypassing the above settings" in the branch protection rule.
- #7 LFS object missing on push — Q4=B intentionally pushes
  pointer-only with `--no-verify`; soft-skipped in tests by the
  fork-only `conftest.py`.
- #8 Subproject not covered by build/test — `pyproject.toml` already
  has `testpaths = ["dimos"]`, no examples/ leak.
- #9 First pre-commit run reformats hundreds of files — none on this
  fork; the upstream code was already format-clean per ruff-format.
- #10 CI Python install is slow — first run paid 11 min, subsequent
  runs benefit from `setup-uv@v6 enable-cache`.

## Reproducing the pipeline on a sibling fork

1. Read `docs/dimos_pipeline_setup_for_agent.md` end-to-end. The Q/D
   decisions are explicit; revisit them for the new context (some
   values are fork-specific, e.g. `Q4` is influenced by the corp LFS
   server's reachability).
2. Open `.cursor/commands/ship.md`. That is the canonical PR flow.
3. Always run `bash scripts/verify.sh` locally before push.
4. Open PRs via REST (not `gh pr create`) per `ship.md` step 7.
5. After branch protection is configured, verify with:
   ```
   gh api repos/<owner>/<repo>/branches/main/protection \
     | jq '{contexts: .required_status_checks.contexts,
            strict: .required_status_checks.strict,
            enforce_admins: .enforce_admins.enabled}'
   ```
   Expected: `contexts: ['verify']`, `strict: true`,
   `enforce_admins: true`. An empty `contexts` array means the
   "Status checks required" search box was ticked but the check was
   never added — you have a paper protection.

## Ownership

- Pipeline-config files (`scripts/verify.sh`, `conftest.py`,
  `.github/workflows/verify.yml`, `.cursor/`) are fork-only and
  maintained by `jiangtao129`. Edits go through `/ship`.
- Upstream workflows (`ci.yml`, `macos.yml`, etc.) are fork-gated; the
  edits to those files are `if: github.repository == ...` guards only,
  cherry-pickable upstream if ever needed.
- This report file is informational; future reports go in
  `docs/pipeline_test_report_<date>.md` alongside this one.
