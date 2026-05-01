# Prethrift — Development Workflow

How a change moves from a requirement or bug report to deployed code, end to end. This is the loop the team actually runs, with the tools wired up in this repo (`.claude/`, `.github/workflows/`, `scripts/ci/`, `scripts/operational/`).

## 0. The shape of the loop

```text
issue → worktree+branch → /plan → /implement → /review → /fix-tests → commit → /commit-push-pr → CI → human review → merge → deploy
```

Each phase has a tool: GitHub for tracking, git worktrees for isolation, Claude commands and agents for the heavy lifting (in `.claude/commands/` and `.claude/agents/`), local hooks and `scripts/ci/` for the local quality gate, GitHub Actions for the shared gate, and CDK for deploy.

---

## 1. Capture: GitHub issue from a requirement or bug

Every change starts as a GitHub issue. The branch name encodes the issue id (`backend-issue-146`, `chore/...`, `fix/...`), so `git log` and `gh pr` stay traceable back to the source of truth.

What goes in the issue:

- **What**: behavior change or bug, in user terms.
- **Why**: business or technical motivation (this maps to the `Why` line in the PR).
- **Scope hints**: which service(s) it's likely to touch (`cart`, `inventory`, `search`, …), whether it's API-shape-changing, whether it touches infra.
- **Out of scope**: what we explicitly will *not* do — keeps the PR small and reviewable per `CLAUDE.md`.

For bugs, include reproduction steps and the smallest failing input. For features, include the API contract or event payload sketch — that's what the planning step will validate against `specs/api/*.yaml` and `packages/domain-events`.

## 2. Isolate: git worktree + branch per issue

We use **git worktrees** so multiple issues can be in flight without stashing or context-switching. The repo's directory naming (`backend-issue-146/` alongside `backend-issue-144/`, `backend-issue-150/`) reflects this — each worktree is a full checkout pinned to its own branch.

The standard path is the **`/new-issue-worktree` Claude command** (see `.claude/commands/new-issue-worktree.md`). From any existing worktree:

```text
/new-issue-worktree <issue number | #n | issue URL>
```

It runs `gh issue view`, refuses on closed issues unless confirmed, creates `../backend-issue-<n>` from a fresh `origin/main` on a new `backend-issue-<n>` branch, runs `npm install` in the new worktree, and emits a paste-ready `/plan` block (issue title, labels, body) for the new Claude session. It refuses if the path or branch already exists — no overwrite, no destructive git.

For the bare case without an issue (spike, scratch worktree), the manual flow still works:

```bash
git fetch origin
git worktree add ../backend-issue-<n> -b backend-issue-<n> origin/main
cd ../backend-issue-<n>
npm install            # workspaces install per worktree
```

Why worktrees over branch-switching:

- `node_modules`, `cdk.out`, `tsbuildinfo` per worktree → no cross-pollination.
- Multiple Claude sessions can run in parallel, each with its own working dir.
- `git status` in any worktree is honest; nothing depends on global stash state.

Cleanup when the PR is merged:

```bash
git worktree remove ../backend-issue-<n>
git branch -d backend-issue-<n>
```

## 3. Analyze and plan with Claude (`/plan`)

Per `CLAUDE.md`, non-trivial changes are **plan-first**. Open Claude in the worktree and run:

```text
/plan <link to issue or summary>
```

The `plan` command (see `.claude/commands/plan.md`) drives:

1. **Repo reconnaissance** — uses the `Plan` / `Explore` agents to map affected boundaries: which `services/<name>/`, which shared `packages/`, which `infra/cdk/lib/*-stack.ts`, which `specs/api/*.yaml`.
2. **Boundary check** against `CLAUDE.md` rules (no cross-service DB reads, contracts in shared packages, OpenAPI is the API source of truth, events are versioned).
3. **Test surface** — unit / integration / contract / e2e split per Testing standard.
4. **Risk and rollback** — what gets deployed, what rolls back, what migrates, what's idempotent.
5. **Output**: a short numbered plan with file paths and the test list, **before** any edits.

For risky or architecturally-novel work, stop here and confirm the plan with a human reviewer. For routine work, proceed straight to implementation.

If the change is Effect-heavy, use `/implement-effect` instead — it loads the `prethrift-effect` skill and the `effect-implementer` + `effect-reviewer` agents which know the `Effect.gen` / `Context.Tag` / `Layer.effect` / `createLambdaRuntime` patterns.

## 4. Implement with Claude (`/implement` or `/implement-effect`)

The implementation step is **plan-driven, not freestyle**. Claude:

- Edits service code under `services/<name>/src/{handlers,domain,events,schemas}` keeping handlers thin.
- Routes new shared code to the right package (`packages/effect-utils`, `packages/domain-errors`, `packages/domain-events`, `packages/aws-effects`, etc. — see the **Architecture map** in `CLAUDE.md`).
- Validates inputs at the boundary with **Zod**, never weakens the domain schema to accept LLM/external slop — normalize first.
- Adds tagged domain errors, never collapses to `Error`.
- Updates the matching OpenAPI spec under `specs/api/*.yaml` when the HTTP shape changes, and runs `npm run generate:types` so frontend and tests stay in sync.
- Adds tests **alongside** the change: unit for domain logic, integration for adapters, contract for APIs/events, e2e only for critical paths.

Two contract-keeping helpers run inside this phase:

- **`/sync-openapi`** — when an HTTP shape changes, diffs the handler's request/response against `specs/api/<service>.yaml`, proposes the spec patch, runs `generate:types` + `openapi:check`, and fails loudly if the handler is the wrong source of truth.
- **`/event-contract <event>`** — when adding or evolving an EventBridge event, walks the producer → bus → consumer chain (schema in `packages/domain-events`, the producer's stream-publisher, EventBridge rule in CDK, every consumer handler, version bump policy) so nothing is forgotten.

When implementation needs a one-off DDB or OpenSearch backfill, **`/backfill-script`** generates a `scripts/migrations/<n>-<slug>.ts` with the repo's standard shape (Effect runtime, dry-run flag, batch + checkpoint, telemetry).

Skills and agents that get pulled in:

- **`prethrift-effect`** skill — the Effect playbook for this codebase.
- **`effect-implementer`** agent — generates Effect-shaped services and Lambda handlers using the repo's conventions.
- **`refactorer`** agent — when the implementation reveals a need to clean up an adjacent boundary (rare; we resist incidental refactors per `CLAUDE.md`).
- **`fast-cli-tools`** skill — pushes Bash usage toward `fd` / `rg` / `ast-grep` / `sd` over `find` / `grep` / `sed`.

While editing, the tooling chain runs in the loop:

- **TypeScript** strict (`npm run typecheck`) — Claude runs this as it goes.
- **ESLint** with the workspace config (`npm run lint`).
- **Prettier** for formatting (`npm run format`).
- **markdownlint-cli2** for any docs touched.
- **syncpack** if dependencies move (`npm run deps:lint`).

## 5. Review (`/review-effect`, `/review-pr`, `/review`, `/security-review`)

Before committing, Claude reviews itself with role-specific agents. We treat these like a pre-flight checklist, not a single pass:

| Command | Agent(s) | What it checks |
| --- | --- | --- |
| `/review-effect` | `effect-reviewer` + `prethrift-effect` skill | Effect idioms: layer composition, deterministic side-effect boundaries, typed errors, handler purity. |
| `/review-pr` | `typescript-reviewer`, `aws-reviewer` | Senior-engineer pass: types, validation, IAM, CDK changes, hidden coupling. |
| `/review` | generic | Sanity pass for non-Effect changes. |
| `/test-effectiveness` | `test-auditor` | Whether new tests verify behavior or just exercise code. |
| `/fix-tests` | `test-auditor` + general | Strengthens or repairs the suite if `/test-effectiveness` finds gaps. |
| `/security-review` | (security skill) | OWASP-style pass on the diff: input validation, authz, secrets, PII. |
| `/simplify` | (simplify skill) | Looks for accidental complexity, dead code, premature abstractions. |
| `/cross-service-check` | grep + `CLAUDE.md` rules | Fails the diff if a service imports from another service's `domain/` or references another service's DDB table — enforces the "no cross-service DB reads" rule. |
| `/idempotency-audit` | static analysis on changed handlers | For changed `services/*/src/handlers/*-handler.ts`, checks each consumes a stable id and either uses `IDEMPOTENCY_TABLE_NAME` or has a documented natural idempotency key. |
| `/cdk-diff-summary` | `cdk diff` digest | Runs `cdk diff` per stack the PR touches, summarizes only IAM, security-group, and stateful-resource (DDB / S3 / Cognito / OpenSearch) deltas, and flags destructive replacements. |

Output of these is iterative — Claude re-edits until each pass is clean. Anything ambiguous gets flagged in the PR description rather than guessed in code (per `CLAUDE.md` "flag uncertainty").

## 6. Local quality gate (pre-push)

The pre-push hook runs `scripts/ci/pre-push-checks.sh`, which delegates to `scripts/ci/validate-changed.sh`. That script:

- Diffs the worktree against `origin/main`.
- For each affected workspace, runs **lint + typecheck + unit tests** only on the changed surface (`npm run validate:changed`).
- Refuses the push if anything fails.

This is the same gate as Phase 7's CI but scoped to the diff, so the round trip stays fast. We do not bypass it (`--no-verify` is forbidden).

## 7. Commit, push, open PR (`/commit-push-pr`)

```text
/commit-push-pr
```

Drives:

1. `git status` / `git diff` / `git log` review.
2. Stages **specific files** (never `git add -A` — secrets risk).
3. Crafts a conventional commit message (`fix(cart): …`, `feat(search): …`) focused on the *why*.
4. Creates the commit (no `--amend`, no `--no-verify`).
5. Pushes the branch with `-u`.
6. Opens the PR via `gh pr create` with the standard body template:

```markdown
## What changed
## Why
## Risk
## Rollback approach
## Test evidence
## Follow-ups
```

That body is mandated by `CLAUDE.md` "PR expectations" — any reviewer should be able to assess blast radius without reading the diff.

## 8. CI on the PR (GitHub Actions)

Three workflows fire on `pull_request` (see `.github/workflows/`):

- **`pr-checks.yml`** — the broad gate. Runs `npm ci`, `install:cdk`, `typecheck`, `build:packages`, `openapi:check`, `api-types:check`, `lint`, `format:check`, `test:unit`. This is the contract the diff has to honor.
- **`classify-tests.yml`** / **`ingest-tests.yml`** — service-targeted integration suites for the classifier and ingest pipelines (Phase 1 / Phase 2 in the workflows section of the architecture doc).
- **`cdk.yml`** — runs on PR + on push to `main`. PR mode synthesizes CDK and diffs against current dev state so reviewers can see infra deltas. Push-to-`main` mode chains into `predeploy.yml` and then deploys.
- **`predeploy.yml`** — bundles Lambda layers, runs final type build, prepares `cdk.out` artifacts.
- **`cleanup-artifacts.yml`** — housekeeping for old CI artifacts.

Anything red blocks the merge.

## 9. Human review and merge

The PR is small, the description is structured, the CI is green — review focuses on judgment calls:

- **Architecture**: did we respect service boundaries? new shared types in the right package? events versioned?
- **API/event shape**: is the OpenAPI / event-schema change the one we want long-term?
- **Risk and rollback**: is the rollback plan real?
- **Observability**: are logs and traces useful for the on-call?

Squash-merge to `main`. The PR title becomes the commit (so it should be PR-quality, not branch-quality).

## 10. Build → deploy

`cdk.yml` on `main` runs **`predeploy` → CDK build → deploy** to the **single shared dev account** (we currently have one dev environment). What gets deployed is selectable via `npm run deploy:service <service>` (script: `scripts/operational/deploy-service.sh`) for targeted iteration, and `npm run deploy` for a full synth+deploy.

Stack scope is per service (`cart-stack.ts`, `inventory-stack.ts`, …) so an inventory change rarely re-deploys cart. Cross-cutting changes (`shared-api-gateway-stack.ts`, `eventbridge-stack.ts`, `auth-stack.ts`) require explicit awareness — the PR description should call them out.

For targeted iteration on dev, **`/dev-deploy <service>`** is the standard path: it checks `git log` on `main` since the last `cdk deploy` to warn if PRs are racing for the shared environment, runs `npm run deploy:service`, then runs the service's e2e suite against dev. After the deploy lands, **`/release-notes`** groups commits since the last deploy tag by service and emits a release-notes block.

Post-deploy validation has two layers:

1. **e2e suites against dev** — `vitest.config.e2e.ts`, `test:e2e:eventbridge`, `test:e2e:complete-workflow` confirm the live system end-to-end.
2. **`/trace-check <traceId | testName>`** — fetches the OpenTelemetry trace for an e2e run from the OTel backend and compares it against the expected-shape spec stored next to the test (`expected-trace.yaml`). Reports missing spans, extra spans, latency outliers, and span attributes that don't match. This makes the *trace shape* a behavior contract — a refactor that accidentally collapses a stage or replaces an event with a sync call is caught here even when unit and integration tests still pass. Used for the golden-path async pipelines (the six workflows in `ARCHITECTURE_OVERVIEW.md`); not maintained for synchronous CRUD where existing integration tests already cover behavior.

---

## Outstanding: ephemeral environments

Today there is **one shared dev environment**. Every PR that needs an integration check shares it, which means:

- Two PRs touching the same stack serialize on the deploy queue.
- Reverting one PR can disturb the other's in-flight test run.
- "Works on dev" is partially a function of who deployed last.

Where we want to go: **per-PR ephemeral environments**, conceptually:

- A CDK stack alias derived from the PR number (e.g. `PrethriftCartStack-PR-146`), with isolated DynamoDB tables, S3 buckets, EventBridge bus, OpenSearch index, and Cognito user pool — or a subset of these scoped via stage prefixes.
- A new GitHub Actions workflow (`pr-deploy.yml`) that, on `pull_request` opened/updated and an `ephemeral` label, deploys the diff to that PR-scoped stage, posts the URLs back as a PR comment, and runs e2e tests against it.
- A teardown workflow on `pull_request` closed that runs `npm run destroy:service` (existing script) for that PR's scope.
- Quotas + budget guardrails (cost-monitoring-stack, account-level limits) so PR environments cannot run away.

Open design questions before we build this:

- **Data**: do ephemeral stacks get empty tables, a sanitized snapshot, or pointers to shared read-only data? Sourcing/classifier work needs sample images; cart needs Shopify test creds.
- **Shared infra**: API Gateway custom domains and Cognito user pools are expensive to provision per PR — likely keep one shared instance with stage-suffixed paths/clients.
- **OpenSearch**: per-PR domains are slow and costly; per-PR indices on a shared domain are likely the right unit.
- **Bedrock / LLM cost**: gate ephemeral e2e runs behind a label so we don't burn LLM credits on every typo PR.
- **EventBridge**: each ephemeral stage needs its own bus or strict source/detail-type prefixing to prevent cross-PR fan-out.

Until that lands, the discipline is: small PRs, prefer unit + integration tests over e2e on dev, coordinate dev deploys in chat, and rely on `cdk diff` in `cdk.yml` PR runs to surface infra surprises before merge.

---

## Tooling cheat-sheet

| Phase | Tool | Command |
| --- | --- | --- |
| Worktree (from issue) | Claude | `/new-issue-worktree <n>` |
| Worktree (bare) | git | `git worktree add ../backend-issue-<n> -b backend-issue-<n> origin/main` |
| Plan | Claude | `/plan` |
| Implement | Claude | `/implement` or `/implement-effect` |
| OpenAPI contract | Claude | `/sync-openapi` |
| Event contract | Claude | `/event-contract <event>` |
| Backfill scaffold | Claude | `/backfill-script` |
| Review | Claude | `/review-effect`, `/review-pr`, `/test-effectiveness`, `/security-review`, `/simplify` |
| Boundary check | Claude | `/cross-service-check` |
| Idempotency audit | Claude | `/idempotency-audit` |
| CDK diff digest | Claude | `/cdk-diff-summary` |
| Local gate | npm | `npm run validate:changed` (auto on push) |
| Type-check all | npm | `npm run typecheck` |
| Lint / format | npm | `npm run lint`, `npm run format` |
| OpenAPI sync | npm | `npm run generate:types`, `npm run openapi:check`, `npm run api-types:check` |
| Tests | npm | `npm run test:unit`, `:integration`, `:e2e` |
| Commit + PR | Claude | `/commit-push-pr` |
| CI | GitHub Actions | `.github/workflows/{pr-checks,cdk,predeploy,classify-tests,ingest-tests}.yml` |
| Targeted deploy (dev) | Claude | `/dev-deploy <service>` |
| Targeted deploy (raw) | npm | `npm run deploy:service <service>` |
| Full deploy | npm | `npm run deploy` |
| Trace shape check | Claude | `/trace-check <traceId \| testName>` |
| Release notes | Claude | `/release-notes` |
| Tear down service | npm | `npm run destroy:service` |
