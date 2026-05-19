# AI-916: openbridge-webcomponents: migrate npm publish to GAR via WIF

- **Ticket:** [AI-916](https://linear.app/blwai/issue/AI-916/openbridge-webcomponents-migrate-npm-publish-to-gar-via-wif)

## What Was Done

`.github/workflows/release.yml` on `blue-water-autonomy` now publishes `@blue-water-autonomy/openbridge-webcomponents` to Google Artifact Registry (`us-central1-npm.pkg.dev/blue-water-autonomy-operations/bwa-npm`) using direct Workload Identity Federation instead of GitHub Packages with `secrets.GITHUB_TOKEN`. This is what lets bob's credential broker (which uses a GitHub App token that `npm.pkg.github.com` categorically rejects) pull the package for power-module's e2e run (AI-918).

Mechanism: the `setup-node` GitHub-Packages registry/scope config and the `NODE_AUTH_TOKEN: secrets.GITHUB_TOKEN` publish auth were removed. A `google-github-actions/auth@v2` step (WIF provider only, **no** `service_account`) plus `setup-gcloud@v2` were added; the publish step now writes a GAR `.npmrc` (registry mapping + `_authToken` from `gcloud auth print-access-token` + `always-auth`) via an `echo` brace-block and runs `npm publish --tag latest`. Job permissions changed from `packages: write` to `id-token: write`. The package-identity / version-stamping step is byte-identical to the prior file. Modern tooling already on the branch (`checkout@v6`, `setup-node@v6`, node 24, `npm ci`, `cache: npm`) is preserved.

## Key Decisions

The ticket flagged two blocking open problems explicitly requiring non-unilateral decisions; both were resolved with the user before implementation:

- **Open Problem 2 (WIF ref allowlist) → tag trigger.** AI-920's live `attribute_condition` (verified from the Done ticket / merged infra PR #227) pins this repo (publisher `repository_id 1239120577`) to `refs/heads/main` **or** `refs/tags/*`. Origin has no `main`, so a push to `blue-water-autonomy` would fail the OIDC exchange. The trigger was changed to **tag push**, gated to the **`bwa-v*`** glob — a BWA-distinct prefix matching html-eslint / AI-917 (`tags: ['bwa-v*']`), chosen so a publish is **not** triggered by upstream `v*` tags or by bob run-artifact tags such as `agentic/meta/agentic/AI-916/r0XXXX`. `bwa-v*` ⊂ `refs/tags/*`, so the already-applied WIF condition allows it with **no infra change** (self-contained in this ticket) — instead of extending AI-920's CEL (cross-repo, blocks first publish) or a `main` cutover (larger coordination). The tag name is only the trigger gate; the version still comes from `package.json`. `workflow_dispatch` is retained but only authenticates when dispatched against a tag ref; this is documented in a workflow comment.
- **Open Problem 1 (version lineage) → publish `2.0.0-bwa.<run>`.** The branch's `packages/openbridge-webcomponents/package.json` is `2.0.0-next.29`; the unchanged stamping yields `2.0.0-bwa.${GITHUB_RUN_NUMBER}`. No `package.json` base change. The `0.0.17` lineage is the dead origin-absent local `main` (out of scope). Power-module's consumer constraint (`^0.0.17-bwa.X` → `2.0.0-bwa.X`) is migrated by AI-918 (still Todo); this ticket does not touch power-module. This is documented in the PR body.

**Process note (skill deviation):** the implement-ticket skill prescribes fresh-context subagents for Phases 2–4. For this single, fully-specified config file (target end-state written verbatim in the spec) carrying cross-conversation decision context, implementation was done in-session by the parent with independent mechanical (Phase 3) and acceptance (Phase 4) verification passes. The verification discipline was kept; only the subagent dispatch was skipped as disproportionate to a ~40-line YAML edit.

## Files Changed

**Implementation:**
- `.github/workflows/release.yml` — trigger (tag push gated to `bwa-v*` + `workflow_dispatch`, no branch push), `permissions` (`id-token: write`, dropped `packages: write`), removed `setup-node` `registry-url`/`scope`, added `google-github-actions/auth@v2` (direct WIF) + `setup-gcloud@v2`, replaced GH-Packages publish with GAR `.npmrc` (echo block) + `npm publish --tag latest`, renamed `Publish to GitHub Packages` → `Publish to GAR`.

**Process artifacts:**
- `.agentic/specs/AI-916-migrate-npm-publish-to-gar-via-wif.md` — implementation spec.
- `.agentic/summaries/AI-916-migrate-npm-publish-to-gar-via-wif.md` — this summary.

No other files changed. No SA key files added (verified). Upstream `@oicl` consumer docs (`docs/getting-started-*.md`, `GettingStarted.mdx`) reference the unrelated upstream package install flow and are correctly untouched.

## Testing

No unit/integration harness applies to a CI publish workflow. Verified within the worktree:

- **YAML validity:** `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release.yml'))"` — passes.
- **Static assertions:** name is `Publish to GAR`; `id-token: write` present and `packages: write` absent; no `registry-url`/`scope`; `google-github-actions/auth@v2` + `setup-gcloud@v2` present with the AI-920 WIF provider; **no `service_account:` YAML key** (only an explanatory comment); `.npmrc` via echo brace-block (no heredoc); no `npm.pkg.github.com` / `NODE_AUTH_TOKEN` / `secrets.GITHUB_TOKEN`; `npm ci` and `npm publish --tag latest` retained; tag trigger present with glob `bwa-v*` (not bare `*`), branch trigger absent; package-identity/version-stamping block byte-identical to fork point `dd46479d`.

**Post-merge / out-of-band verification** (ticket "Done when" items #4, #5, #7 — require the live repo + GCP + power-module, not executable from the worktree):
- Push a `bwa-v*` tag (e.g. `bwa-v2.0.0`) to `Blue-Water-Autonomy/openbridge-webcomponents` and confirm the workflow authenticates via WIF and publishes. (No `bwa-v*` tags exist on origin yet; the existing `agentic/.../r05009` artifact tag correctly does not match.)
- `gcloud artifacts versions list --repository=bwa-npm --location=us-central1 --project=blue-water-autonomy-operations --package=openbridge-webcomponents` shows the new `2.0.0-bwa.<run>` version.
- AI-918 updates power-module's constraint to `2.0.0-bwa.X` and its e2e resolves/installs the package from GAR.
