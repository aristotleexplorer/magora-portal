# magora-portal repo harness — design (Spec 1 of 2)

- **Date:** 2026-10-15
- **Status:** draft rev 3 (after Codex rounds 1–2), awaiting Gemini review, then David's review
- **Plan tasks:** H1, H2 of the Magora Consolidated Action Plan (2026-10-09, §08 repo audit)
- **Gate:** G2, 2026-10-30 — harness, PR flow and isolated preview ready before app or site coding
- **Companion:** Spec 2 (staging, plan task E1) is a separate document and is not designed here

This document specifies; it implements nothing. No file, setting or app described here exists yet.

## 1. Goal

Give magora-portal a pull-request flow that two maintainers (David, Noah Waldron) and, later, fork
contributors can work in, where checks that matter actually block a bad merge and every control
lives in GitHub or in files in this repo.

**Success criteria (Spec 1 is done when all hold):**

1. Each required check has been shown to **fail on a defect it exists to catch** before protection
   is armed (§6 step 6).
2. After protection is armed, the gate has been shown, one cause at a time, to **block** on a
   missing approval and on each required check failing, and to **admit** an ordinary approved PR
   (§6 step 8).
3. A fork PR has been observed to receive CI and no secrets; a Dependabot PR has been observed to
   pass through the gate.
4. Both maintainers' agents (Claude Code and Codex CLI, on macOS and Windows) have been observed
   to pick up `AGENTS.md` (§6 step 9).

The isolated-preview half of G2 belongs to Spec 2.

## 2. Constraints

- **Two maintainers, two Claude accounts, two OSes.** David develops on macOS, Noah on Windows.
  Both run Claude Code and the Codex CLI.
- **Controls live in GitHub or repo files.** David's personal hooks in `claude-config` (the
  push-review marker gate, the pre-push hook) never reach Noah or forks. They remain optional
  personal aids and are not part of this design.
- **No self-posted gating status.** The approving review comes from the other maintainer, never
  from a status the author's own credential posts.
- **Fork safety.** Every PR-triggered workflow uses `pull_request`, never `pull_request_target`.
  The only other trigger in this design is a weekly `schedule` on the OSV scan, which runs on
  `main`, executes no PR code and uses no secret. No `push` triggers. Actions are pinned by full
  commit SHA. `GITHUB_TOKEN` is least-privilege per workflow. No secret is reachable from a fork
  PR's CI.
- **Proof before protection.** Each check is proven to fail on its target defect before
  protection is armed; enforcement is then proven after arming (§6).
- **The repo is already public** (`visibility: public`, measured 2026-10-15) and has no LICENSE.

### 2.1 Trust boundary (what the gate does and does not attest)

A `pull_request` workflow runs the PR's own copy of the workflow YAML. A PR can therefore weaken
a required job (make it skip, or make it always succeed) and still satisfy its context; GitHub
counts a skipped job as passing. Branch protection attests that a check with the right name
reported success, not that the original command ran. **The required human review is the control
for that**: a reviewer must inspect any change to a *control file* — `.github/**`,
`scripts/run-tests.mjs`, `scripts/ci/**`, `.github/lint-baseline.json`, the migration grandfather
list, `eslint.config.js`, `CODEOWNERS`, `AGENTS.md` — and the PR template asks the author to
declare such changes. The same applies to `gemini-review.yml`: checking out the base SHA protects
the reviewer script, not the workflow YAML, so a same-repo PR can alter the YAML and reach the
Gemini key on its own run. Same-repo PRs come only from the two maintainers, so this is accepted
and recorded, not prevented.

### 2.2 Fork previews (Vercel)

Vercel builds a preview for every PR, and Preview-scoped env vars include `PROVISION_SECRET`,
`ANTHROPIC_API_KEY` and the production Supabase URL and anon key (§3). Git Fork Protection is on,
so a fork PR's preview is not built until a team member authorizes it — but **authorizing one
runs fork code with those secrets against production**. Until Spec 2 moves Preview to staging:

- **No maintainer authorizes a fork-PR Vercel deployment.** Fork PRs are reviewed from CI and the
  diff only. `AGENTS.md` and `CONTRIBUTING.md` state this.
- Git Fork Protection is read back in §6 step 4 and must stay on; turning it off is a control-file
  change in spirit and needs both maintainers.
- Same-repo (maintainer) previews keep hitting production, as they do today.

## 3. Current state (measured 2026-10-15 unless noted)

| Fact | Evidence |
|---|---|
| `origin/main` = local `main` = `1fae773` | `git fetch` then `git log origin/main` |
| No `.github/` directory, no workflows | `ls .github` → absent |
| No LICENSE, CONTRIBUTING, SECURITY, CODEOWNERS, AGENTS.md, CLAUDE.md | top-level listing |
| Merge methods: merge, squash and rebase all allowed; default branch `main`; public | `gh api repos/magora-project/magora-portal` |
| `package.json` scripts: `dev`, `vendor:build-doc`, `prebuild`, `build`, `lint`, `preview`; no `test` | `package.json` |
| `prebuild` runs `scripts/vendor-build-doc.mjs`, which fetches another repo's mutable `release` ref and rewrites `src/generated/build-doc.js`, falling back to the committed copy on fetch failure; so `build` is network-dependent | read the script (Codex round 1) |
| Plain JavaScript, no TypeScript in the app; Deno TypeScript in `supabase/functions/`, importing `https://esm.sh/@supabase/supabase-js@2` (unpinned minor) | file listing; Codex round 1 |
| Six harnesses `api/**/*.test.mjs`; all six exit 0 locally; only `proxy.test.mjs` uses `fetch`, and stubs it. Imported modules contain network-capable code, so "no network" is not guaranteed | ran each with `node`; Codex round 1 reproduced on Node 22 |
| Other `scripts/*.mjs` (backfills, `verify-reference-audio`, `eval-insight-models`) hit networks, databases or paid models; none matches `*.test.mjs` | Codex round 1 |
| ESLint matches only `**/*.{js,jsx}` with browser globals, so `.mjs` files and Deno `.ts` are not linted | `eslint.config.js` |
| Lint: 29 errors in 9 files — 16 `no-undef` (`process` in Node files under `api/`), 7 `react-hooks/static-components`, 3 `react-hooks/set-state-in-effect`, 2 `react-refresh/only-export-components`, 1 `no-unused-vars` | `npx eslint . -f json` after `npm ci`; reproduced by Codex |
| 17 files under `api/` and `scripts/` carry `/* global process */` (14) or `/* global process, fetch */` (3) comments; adding Node globals turns these into 20 `no-redeclare` errors | grep; Codex reproduced in memory |
| `vite.config.js` proxies dev `/api` to `https://magora-portal.vercel.app` (production) | `vite.config.js` |
| 32 migrations in `supabase/migrations/`, none with a rollback file | listing |
| Edge Functions `provision-node`, `storage-upload` | `supabase/functions/` |
| Vercel project `magora-portal` (team `noah-waldron-s-projects`) is linked to `magora-project/magora-portal`, production branch `main`, with **Git Fork Protection on** (`gitForkProtection: true`): a fork PR's preview is not built until a team member authorizes it | `vercel api /v9/projects/magora-portal`, fields projected, no values read |
| Preview-scoped env vars (names only): `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` (production project), `ANTHROPIC_API_KEY`, `EBIRD_API_KEY`, `PROVISION_SECRET`, `XENO_CANTO_API_KEY`. `SUPABASE_SERVICE_ROLE_KEY` and `CRON_SECRET` are production-only | same call |

**Not yet measured** (pre-flight, §6 step 1): both maintainers' org roles and repo permissions;
installed GitHub Apps (`gh` token lacks `admin:org`); the org/repo fork-workflow approval policy;
whether `vite build` succeeds with no env vars; whether `deno check` succeeds on both functions;
how the Supabase CLI treats a `supabase/migrations/rollbacks/` subdirectory.

## 4. Decisions taken in the brainstorm

| # | Decision | Rejected alternatives |
|---|---|---|
| D-1 | Two specs: this harness spec, and a separate staging spec (E1). Previews stay production-backed until Spec 2 lands; this spec says so rather than fixing it. | Moving the Vite-proxy repoint into Spec 1. |
| D-2 | **Trunk flow.** Feature branches PR straight to `main`; no long-lived `dev`, no promotion step, no `main → dev` sync. Each PR's Vercel preview is where it is checked. Promotion is selective because each PR merges independently. | method-project's whole-`dev` promotion; a disposable integration `dev`; cherry-pick release PRs. A shared integration branch can be added later if needed. |
| D-3 | **Squash merge only**, delete branch on merge. | Merge-commit only (method-project's promotion rule). |
| D-4 | **Reviewer matrix recycled in two layers** (§5.3): Layer 1 on the PR, Layer 2 as `AGENTS.md` guidance for agents before pushing. | Bots as required checks; Layer 2 as hooks (cannot reach Noah or forks). |
| D-5 | **Lint: fix the config, then ratchet.** Correct globals for Node files, bring `.mjs` into lint, record per-rule counts as a baseline; CI fails if any rule's count rises. The 13 remaining `src/` findings are fixed in their own PRs. | Fix all findings in the harness PR; inline `eslint-disable` with TODOs. |
| D-6 | LICENSE does not block G2; it blocks inviting fork contributors. | — |

## 5. Components

### 5.1 CI — `.github/workflows/ci.yml`

- Trigger: `pull_request` (all branches, types `opened`, `synchronize`, `reopened`,
  `ready_for_review`). No path filters — a filtered workflow leaves required checks pending on
  PRs it skips. Every job runs on every PR, including drafts, docs-only and Dependabot PRs.
- `permissions: contents: read`. No secrets referenced.
- Node version from a checked-in `.nvmrc`; `npm ci`.
- Every check is an npm script implemented in Node (no bash), so it runs the same on Windows.

| Job (context name) | Command | Notes |
|---|---|---|
| `lint (ubuntu-latest)`, `lint (windows-latest)` | `npm run lint:ratchet` | `scripts/ci/lint-ratchet.mjs` runs ESLint with JSON output and compares per-rule counts to `.github/lint-baseline.json`. Fails on any increase. A decrease prints a reminder to lower the baseline in the same PR; it does not fail. **Fails closed** on anything that is not a counted rule violation: a fatal diagnostic or `ruleId: null` (parse errors), ESLint failing to start or exiting for a config error, output that is not valid JSON, or a missing or malformed baseline. ESLint's ordinary "lint errors found" exit is expected and tolerated; the ratchet tells it apart from an execution failure. Its own self-test covers each fail-closed case. |
| `test (ubuntu-latest)`, `test (windows-latest)` | `npm test` | `scripts/run-tests.mjs` finds every `*.test.mjs` outside `node_modules`, runs each with `node` as a child process, and fails if any exits non-zero, if any child fails to launch, or if it found zero files. The runner has its own `scripts/run-tests.test.mjs` covering those three failure modes, so it tests itself. |
| `build` | `npm run build` | Ubuntu only. Must pass with no env vars. `prebuild` fetches the build doc over the network and falls back to the committed copy, so a fetch failure does not fail CI; recorded as existing behaviour. |
| `deno` | `deno check` and `deno lint` over `supabase/functions/` | Deno version pinned. If pre-flight shows `deno check` fails on current code, the job starts as `deno lint` only and the spec records it. |
| `migrations` | `npm run check:migrations` | §5.5. |

Exact context names are confirmed from the proof PR (V0), not assumed from this table. Matrix
jobs set `strategy.fail-fast: false`, so one OS failing never cancels the other before it reports.

ESLint config change (D-5): a second config block applying `globals.node` to `api/**`,
`scripts/**` and `**/*.mjs`, widening the file pattern to include `.mjs`, **and deleting the 17
now-redundant `/* global … */` comments**. The baseline is recorded after this, from a fresh run;
it must contain no `no-redeclare` and no `no-undef` entries.

### 5.2 Security

Adapted from `~/method-project/.github/workflows/`, with every Action re-pinned by SHA (the
method-project OSV workflow is tag-pinned) and every helper/config file it reads included:

- `semgrep.yml` on `pull_request`.
- `osv-scan.yml` on `pull_request` **and** a weekly `schedule` (method-project's runs on schedule
  only), plus its `osv-scanner.toml`.
- `.github/dependabot.yml` for `npm` and `github-actions`, with a `cooldown`, and
  `labels: ["standard"]` so every Dependabot PR arrives with exactly one depth label (§5.3).

Semgrep and OSV are **advisory for G2** and are not in D1's required set. Promoting either to
required is a later, separate decision once their false-positive rate on real PRs is known.

### 5.3 Review

**Layer 1 — on every PR, whoever opens it.**

| Reviewer | Role | Blocking? |
|---|---|---|
| A maintainer other than the author (CODEOWNERS) | The gate. One approval; stale approvals dismissed on new commits. Inspects control-file changes (§2.1). | **Yes** |
| CodeRabbit (GitHub App, fleet `.coderabbit.yaml` baseline) | Analyzer-running reviewer | No |
| Codex GitHub App | Cross-vendor correctness | No |
| Gemini (`gemini-review.yml`) | Cross-vendor complement lens, comment-only | No |

GitHub never lets an author approve their own PR, so a maintainer's PR needs the other
maintainer; a contributor's or Dependabot's PR needs either.

**Gemini workflow.** Triggers `pull_request` types `opened`, `synchronize`, `reopened`,
`ready_for_review`, `labeled`. Runs only when all hold: non-draft; same-repo
(`head.repo.full_name == github.repository`); neither the triggering actor nor the PR author
(`pull_request.user.login`) is `dependabot[bot]` — Dependabot PRs are excluded however they are
labelled; label `subtle-correctness` present; and, on a `labeled` event, the label just added is
`subtle-correctness` (other label additions do not re-run it). Gemini is reachable on Dependabot
PRs only if a maintainer opts in later; that is out of scope. Checks out the **base** SHA for its
helper script, mirroring method-project. The helper script it runs is added in PR B; until PR B
is on `main`, the workflow cannot run (bootstrap, §6). The key is a Magora-owned Actions secret.
**If open decision §8.3 (who owns and pays for the key) is unresolved when PR B is ready, Gemini is
deferred:** `gemini-review.yml` is left out of PR B and added in a follow-up PR once the key
exists. Gemini is advisory, so G2 does not depend on it; Layer 1 then runs with CodeRabbit and the
Codex App.

**Depth labels.** `mechanical`, `standard`, `subtle-correctness`, created in §6 step 4.

**`label-gate.yml`.** Requires exactly one depth label.
- Implemented as a job whose own pass/fail is the check, not a posted commit status: a fork
  PR's `GITHUB_TOKEN` is read-only and cannot write statuses.
- Triggers `pull_request` types `opened`, `synchronize`, `reopened`, `labeled`, `unlabeled`, so
  removing a label re-evaluates.
- Reads the PR's **current** labels from the API (`permissions: pull-requests: read, issues:
  read`) at evaluation time, not the event payload.
- `concurrency` group keyed on workflow, PR number **and head SHA**, with
  `cancel-in-progress: false`. Runs for an older head report against that older SHA and cannot
  touch the current head's check. Runs for the same head queue rather than overlap; GitHub may
  cancel a queued run when a newer one is queued, and the newer one then runs and reads current
  labels. A cancelled run is not an accepted conclusion, so if the latest run on the head is ever
  `cancelled`, recovery is to re-run the job or toggle a label; `CONTRIBUTING.md` says so.
- Fork authors cannot add labels; the reviewing maintainer applies the depth label.
- A first-time contributor's fork runs wait for workflow approval regardless of labels; a
  maintainer approves the run after inspecting the change.

**PR template** (`.github/pull_request_template.md`): depth checkbox, what changed, test
evidence, migration checklist, and "this PR changes control files: yes/no" (§2.1).

**Layer 2 — in `AGENTS.md`, followed by whichever agent is working before it pushes.** This is
the claude-config Grid A matrix, unchanged in substance:

| Depth | Before push |
|---|---|
| `mechanical` | Nothing beyond CI. |
| `standard` | Codex CLI, correctness question. |
| `subtle-correctness` | Codex CLI on correctness, plus a Fable or Opus subagent on the complement lens (completeness, executability, can these tests fail, what did the fixes break). |
| Any depth, author was Codex | Route the review to an Anthropic model instead. |

An empty reviewer completion is an unrun check, not a pass. Layer 2 is discipline, not a control;
Layer 1 is the control.

**Left out on purpose:** the marker gate, the self-posted `codex-review` status, the tier
classifier, the review ledger, auto-merge, wait-for-pr-age, the orchestrator, owner-approval,
issue-closer.

### 5.4 Shared instructions and contributor files

- `AGENTS.md`: the single source for both maintainers' agents and for contributors' agents.
  Contents: commands (`npm ci`, `npm test`, `npm run lint:ratchet`, `npm run build`), the trunk
  flow, depth labels and how to choose one (any PR touching `supabase/migrations/`, `api/` auth
  paths or a control file is `subtle-correctness`), Layer 2, the control-file rule (§2.1), the
  migration rules (§5.5), the
  warning that local `vite dev` and previews hit production until Spec 2, and the rule that agents
  never push without their operator's approval. Codex CLI reads `AGENTS.md` natively.
- `CLAUDE.md`: a thin pointer to `AGENTS.md`, because Claude Code reads `CLAUDE.md`.
- `CONTRIBUTING.md`: fork workflow, depth labels, updating a behind branch (which re-runs CI and
  dismisses approvals), how to run checks locally on Windows and macOS. Licensing and DCO/CLA
  marked pending (§8).
- `SECURITY.md`: private reporting via GitHub security advisories.
- `.github/CODEOWNERS`: `* @<david-handle> @<noah-handle>`, handles confirmed in pre-flight.
  CODEOWNERS is read from the base branch, so it governs PRs opened after PR B merges.
- `doc-size-ratchet.yml` on `AGENTS.md` and `CLAUDE.md`, adapted from method-project with its
  helper script and config included in PR B. **Advisory** (not required): it guards prose size,
  not correctness.

### 5.5 Migration rule, enforced by CI

- `scripts/ci/check-migrations.mjs` fails if a migration file added in the PR has no paired
  rollback file. It discovers only top-level forward migrations (`supabase/migrations/*.sql`) and
  never treats `.grandfathered` or anything under `rollbacks/` as a migration. The 32 existing migrations are grandfathered by name in
  `supabase/migrations/.grandfathered` (a control file, §2.1).
- Pairing mirrors method-project: `supabase/migrations/<stem>.sql` pairs with
  `supabase/migrations/rollbacks/<stem>.rollback.sql`. Pre-flight confirms, with a named Supabase
  CLI version, that the CLI ignores the `rollbacks/` subdirectory; if it does not, rollbacks move
  to `supabase/rollbacks/` and this section is amended before the plan.
- `AGENTS.md` states: expand/contract for anything a running deploy reads; a migration is never
  applied until its PR is green at its final head. **Where** migrations are applied is Spec 2.

### 5.6 Repo settings (each written in §6 step 4 and read back)

- Squash merge only; auto-delete head branches.
- Dependabot alerts and security updates on.
- Fork-workflow approval: require approval for first-time contributors (at least). The current
  org/repo policy is measured in pre-flight; the write is made only if it is weaker.

### 5.7 Branch protection on `main` (D1)

- Required status checks: exactly the contexts proven in §6 step 6, by exact name from V0,
  **each bound to the GitHub Actions app as its source** so no other credential can satisfy it.
- Require branches up to date before merge.
- One approving review, from CODEOWNERS; dismiss stale approvals.
- Require conversation resolution.
- Bypass holder: open decision (§8). Until decided, admins are **not** exempted.

## 6. Rollout

Order follows `claude-config/scripts/fleet-pr-pipeline/repo-onboarding-checklist.md`, adapted for
two maintainers. Its scripts are hard-wired to owner `bluemtnswalking1` and a solo owner, so the
procedure is reused, not the scripts. **Every push, settings write and app install waits for
David's explicit approval.**

1. **Pre-flight (read-only).** Measure both maintainers' org roles and repo permissions; list
   installed GitHub Apps; read the fork-workflow approval policy; run `vite build` with no env
   vars; run `deno check` on both functions; confirm Supabase CLI behaviour on the full layout
   (`rollbacks/` subdirectory and the `.grandfathered` file) in a scratch project, recording the
   CLI version; confirm both GitHub handles.
2. **Human-only steps (org owner).** Install CodeRabbit and the Codex App on `magora-project`,
   scoped to `magora-portal`. If §8.3 is decided, create the Magora-owned Gemini key and store it
   as an Actions secret; otherwise Gemini is deferred (§5.3).
3. **PR A — CI foundation, reviewed by Noah.** ESLint config fix and `global`-comment removal,
   lint baseline and ratchet, test runner and its self-test, migration check and grandfather list,
   `ci.yml`, `.nvmrc`. Acceptance before merge: every `ci.yml` job green on PR A's final head;
   Noah's approval on that head. Merged before PR B.
4. **Settings and labels (approved writes, each read back).** Create the three depth labels.
   Write §5.6 settings. Read each back via the API and record the response. Read back Vercel
   `gitForkProtection` (must be `true`, §2.2).
5. **PR B — review, security, contributor files, reviewed by Noah.** Security workflows and
   `osv-scanner.toml`, `dependabot.yml`, `.coderabbit.yaml`, `gemini-review.yml` and its helper,
   `label-gate.yml`, PR template, `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `SECURITY.md`,
   `CODEOWNERS`, doc-size-ratchet and its helper. **Bootstrap limitation:** workflows that read
   base-SHA helpers (Gemini, doc-size-ratchet) cannot run on PR B itself, because the helpers are
   not on its base; PR B is reviewed by hand for those, and they are proven in step 6. Acceptance:
   label-gate and every `ci.yml` job green on the final head; Noah's approval on that head.
6. **Pre-arm proof (no protection yet), on a throwaway proof PR at `subtle-correctness`:**
   - V0: record every check's exact context name.
   - V4: CodeRabbit reviewed this head. V5: the Codex App engaged. Gemini commented.
   - V6: CI green. V7: Dependabot alerts and security updates live.
   - **Each required check fails on its target defect**, per the proof matrix below: one defect
     per commit, reverted after each, and the evidence recorded per context name at that head.
   - Rapid label toggling and two quick pushes leave the current head with a conclusive
     label-gate result.
   - From a test fork: CI runs; Gemini's job is skipped. Gemini is the only workflow that
     references a secret, so its skipped job on the fork run is the evidence that no secret was
     reachable. Label-gate passes once a maintainer labels the PR.
   - A real Dependabot PR (or the first one to arrive): arrives labelled `standard`, CI runs,
     Gemini skips — including after a maintainer adds `subtle-correctness` to it.

   **Proof matrix** (each row's defect must turn red exactly the contexts in its "expected red"
   cell, and every required context outside that cell must stay green):

   | Defect commit / action | Expected red contexts |
   |---|---|
   | New lint error in a `src/` file | `lint (ubuntu-latest)`, `lint (windows-latest)` |
   | Syntax error in a linted file (ratchet fail-closed) | both `lint` contexts and `build` |
   | Failing assertion in one harness | `test (ubuntu-latest)`, `test (windows-latest)` |
   | Broken import in `src/` that only `vite build` resolves | `build` |
   | Deno defect matching the chosen mode (type error if `deno check` runs; lint violation if lint-only) | `deno` |
   | New migration without rollback | `migrations` |
   | Labels: zero, then two, then remove the only one (API actions, not commits) | `label-gate` |
7. **D1 — arm protection** with only the contexts proven in step 6, bound to GitHub Actions.
   Rollback if any later step fails: remove the offending required context (onboarding checklist
   §5), never disable protection wholesale.
8. **Post-arm enforcement proof, isolating each cause:**
   - V1: read protection back and compare with §5.7.
   - V2a: all checks green, no approval → merge blocked.
   - V2b: for each row of the proof matrix: push that defect (or apply that label action), have
     the other maintainer approve **that head** (a push dismisses earlier approvals), confirm
     every required context outside that row's expected-red cell is green, and record that the
     merge is blocked at that head.
     Revert, and repeat for the next row.
   - V3: the first ordinary PR after D1 (a small real change, approved by the other maintainer)
     merges normally. The proof PR is closed unmerged, because merging it would deploy to
     production.
   - Dependabot under protection: a Dependabot PR is approved by a maintainer, passes every
     required context, and merges normally.
9. **Agent smoke check.** On each maintainer's machine, start a fresh Claude Code session and a
   fresh Codex CLI session in the checkout and ask: "What depth label does a PR that adds a
   Supabase migration get, and what must it include?" **Pass** = the answer names
   `subtle-correctness` and a paired `rollbacks/<stem>.rollback.sql` file. Record pass/fail for
   all four (2 agents × 2 machines).
10. **Tell the Magora Orchestrator session** the depth-label convention for the Software Roadmap.

## 7. Risks

| Risk | Mitigation |
|---|---|
| A required check that never reports wedges every PR | No path filters; require only contexts observed in step 6; per-context rollback |
| One maintainer unavailable → nothing merges | Decision §8.2 names the bypass holder(s), configured as a **named bypass allowance on the pull-request review requirement** — not by toggling protection off and on, which can be forgotten. A bypass merge still needs green checks and is recorded in the PR with the reason. Until §8.2 is decided there is no bypass: an urgent fix waits for the other maintainer. |
| A maintainer authorizes a fork preview, running fork code with Preview secrets against production | §2.2 rule in `AGENTS.md` and `CONTRIBUTING.md`; Git Fork Protection read back; removed by Spec 2 |
| A PR weakens a required job in its own YAML | Human review of control files (§2.1); template declaration |
| Fork PR reaches a secret | `pull_request` only; Gemini gated to same-repo, non-Dependabot; proven in step 6 |
| Lint ratchet hides new debt in one rule by fixing another | Ratchet is per rule, not total |
| Test runner passes vacuously | Fails on zero files and launch errors; self-tested; sabotage-proven in step 6 |
| `build` depends on a network fetch | Falls back to the committed copy; recorded, not changed here |
| Deno's `esm.sh` import is unpinned | Recorded; pinning is out of scope (a functions change) |
| Previews and local dev still hit production | Documented in `AGENTS.md`; fixed by Spec 2 |

## 8. Open decisions for David and Noah (not decided here)

1. Portal license, and DCO versus CLA. Blocks inviting fork contributors, not G2.
2. Who holds the emergency bypass on `main` (configured as a named bypass allowance, §7).
3. Who creates and pays for the Magora Gemini key.
4. (Spec 2) the staging domain and who is billed for the staging Supabase project.
5. Whether the PWA-versus-native decision changes the repo layout. The Field App repo comes later,
   from a harness kit extracted after G2.

## 9. Out of scope

Everything in Spec 2: the baseline migration for the five production-only tables, the staging
Supabase project and Fly worker, Vercel Preview env vars, repointing the Vite proxy, the CI
isolation check, RLS tests. Also fixing the 13 remaining `src/` lint findings (10 React-hooks,
2 react-refresh, 1 unused variable) in separate PRs, and pinning the Deno `esm.sh` import.

## 10. Review log

| Round | Reviewer | Result | Disposition |
|---|---|---|---|
| 1 | Codex CLI (`codex exec`, 102,798 tokens) on rev 1 | REWORK, 11 findings (6 P1, 4 P2, 1 P3) | All 11 accepted after checking against the repo and method-project: dropped `push` trigger; label-gate events, live labels and concurrency; split pre-arm proof from post-arm enforcement with per-cause isolation; enumerated helper files, OSV trigger and pins, bootstrap limitation; PR A/B reviewer, acceptance and merge order; settings writes and readbacks, Actions source binding; trust boundary §2.1; Dependabot label and Gemini exclusion; `global`-comment removal; agent smoke check and Supabase CLI proof; script list and lint counts. |
| 2 | Codex CLI on rev 2 (fix-verification prompt) | REWORK, 6 findings (1 P1, 4 P2, 1 P3); 7 of 11 round-1 items closed, 3 partial, 1 not closed | All 6 accepted: per-context proof matrix with `fail-fast: false`, Deno mode matched, label actions replayed separately, fresh approval per post-arm case; ratchet fails closed on fatal/null-rule/invocation/baseline errors; label-gate concurrency keyed on head SHA without cancel-in-progress, plus recovery; Gemini excludes Dependabot-authored PRs and non-`subtle-correctness` label events; Dependabot proven under armed protection; smoke check has a fixed question and pass rule (and `AGENTS.md` now states the depth rule it tests). |
| 3 | Gemini (`gemini-3.1-pro-preview`, `gemini_review.py`, rc 0, `finishReason=STOP`) on rev 3, fitness/executability/human-factors prompt | BLOCK, 6 findings (1 P0, 3 P1, 2 P2) | P0 "fork PRs auto-deploy production-connected previews": premise refuted by measurement (Git Fork Protection is on, so fork previews need authorization), but the conclusion partly holds — an authorized fork preview gets Preview secrets including `PROVISION_SECRET` — so §2.2 added (no fork-preview authorization until Spec 2; read-back). Two-spec split kept, pending David. P1 proof-matrix contradiction: fixed. P1 Gemini key undecided: Gemini deferred if §8.3 is open. P1 security-check criteria: Semgrep/OSV advisory for G2. P2 bypass by toggling: replaced with a named bypass allowance. P2 "cut V2b": **contradicts Codex round 2** (which required per-check post-arm proof); surfaced to David, not decided. |
