# AI-916: openbridge-webcomponents: migrate npm publish to GAR via WIF

- **Ticket:** [AI-916](https://linear.app/blwai/issue/AI-916/openbridge-webcomponents-migrate-npm-publish-to-gar-via-wif)
- **Title:** openbridge-webcomponents: migrate npm publish to GAR via WIF

## Objective

`@blue-water-autonomy/openbridge-webcomponents` is currently published to GitHub Packages (`npm.pkg.github.com`) by `.github/workflows/release.yml` on the `blue-water-autonomy` branch, authenticated with the in-workflow `secrets.GITHUB_TOKEN`. bob's credential broker authenticates with a GitHub App installation token, which `npm.pkg.github.com` categorically rejects (GitHub Staff confirmed). bob therefore cannot pull this package for power-module's e2e run (AI-918). Google Artifact Registry is the only path bob can authenticate against for npm.

This change repoints the publish to GAR (`us-central1-npm.pkg.dev/blue-water-autonomy-operations/bwa-npm`) using direct Workload Identity Federation — no long-lived credentials and no service account. The WIF provider and IAM bindings were created and applied by AI-920 (Done, infrastructure PR #227).

## Approach

Modify only `.github/workflows/release.yml` on this branch (`blue-water-autonomy`). The branch already carries modern tooling (`actions/checkout@v6`, `actions/setup-node@v6`, node 24, `npm ci`, `cache: npm`); keep it. Swap the auth/registry mechanism only, preserving the package-identity and version-stamping steps verbatim.

**Two open problems flagged by the ticket were resolved with the user before implementation:**

### Open Problem 2 — WIF ref allowlist → use a tag trigger (DECIDED)

AI-920's live `attribute_condition` (verified from the Done ticket / merged PR #227):

```
assertion.repository_owner == 'Blue-Water-Autonomy' && (
  ((assertion.repository_id == '1239120577' || assertion.repository_id == '1090507958') &&
   (assertion.ref == 'refs/heads/main' || assertion.ref.startsWith('refs/tags/')))
  || assertion.repository_id == '923037232'
)
```

openbridge-webcomponents is repo `1239120577` — a **publisher**, ref-pinned to `refs/heads/main` **or** `refs/tags/*`. A `push` to `blue-water-autonomy` produces `ref=refs/heads/blue-water-autonomy`, which **fails** the OIDC exchange. `refs/tags/*` is already allowed.

**Decision:** trigger publishing on **tag push**, gated to the `bwa-v*` glob. Any `refs/tags/*` satisfies the already-applied WIF condition with no infra change, but the workflow only fires on `bwa-v*` tags — a BWA-distinct prefix matching html-eslint / AI-917 (`tags: ['bwa-v*']`), chosen so a publish is **not** triggered by upstream `v*` tags or by bob run-artifact tags such as `agentic/meta/agentic/AI-916/r0XXXX` (the over-trigger footgun). `bwa-v*` ⊂ `refs/tags/*`, so WIF still allows it. The tag name is only the trigger gate — the published version is still derived from `package.json` (`2.0.0-bwa.${run}`), not the tag. Branch-push on `blue-water-autonomy` is removed as a trigger. `workflow_dispatch` is retained as a manual convenience; note it only authenticates when dispatched against a **tag** ref (GitHub allows `workflow_dispatch` `ref` to be a tag via API/CLI) — a dispatch from the branch will fail at the WIF step, which is expected and acceptable per the user's decision.

### Open Problem 1 — version lineage → publish `2.0.0-bwa.<run>` (DECIDED)

`packages/openbridge-webcomponents/package.json` on this branch is `2.0.0-next.29`. The existing stamping (`base=version.split('-')[0]`) yields base `2.0.0` → published `2.0.0-bwa.${GITHUB_RUN_NUMBER}`. The `0.0.17` lineage belongs to the dead, origin-absent local `main` (explicitly out of scope).

**Decision:** publish `2.0.0-bwa.<run>` with **no change** to the `package.json` base. Power-module's consumer constraint migration (AI-918, still Todo) updates `^0.0.17-bwa.X` → `2.0.0-bwa.X`. This is documented in the PR and is cross-repo coordination owned jointly with AI-918 — this ticket does not modify power-module.

### Pattern notes

- `.npmrc` is written with an `echo` block, **not** a heredoc — heredoc indentation inside a YAML `run:` block is fragile and broke the AI-920 smoke.
- No `service_account:` line under any circumstance (direct WIF; AI-920 smoke-passed SA-less).
- `--access`-style flags are npm-registry-specific and not applicable to GAR; the existing workflow has none, so nothing to drop there.
- `npm publish --tag latest` semantics preserved.

## Files to Create or Modify

| File | Change |
|------|--------|
| `.github/workflows/release.yml` | Rewrite trigger (tag push + workflow_dispatch), permissions (`id-token: write`, drop `packages: write`), setup-node (drop `registry-url`/`scope`), add `google-github-actions/auth@v2` (WIF, no SA) + `setup-gcloud@v2`, replace GH-Packages publish step with GAR `.npmrc` (echo block) + `npm publish --tag latest`. Rename workflow `Publish to GitHub Packages` → `Publish to GAR`. |
| `.agentic/specs/AI-916-migrate-npm-publish-to-gar-via-wif.md` | This spec (new). |
| `.agentic/summaries/AI-916-migrate-npm-publish-to-gar-via-wif.md` | Summary (Phase 5, new). |

## Key Interfaces and Types

Not applicable (CI workflow YAML, no code interfaces). Target end-state of `.github/workflows/release.yml`:

```yaml
name: Publish to GAR

on:
  push:
    tags:
      - "bwa-v*"
  workflow_dispatch: {}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: "24"
          cache: "npm"

      - name: Authenticate to Google Cloud (direct WIF)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/1066481849278/locations/global/workloadIdentityPools/github-actions/providers/github-oidc-bwa-npm
          # NO service_account — direct WIF (smoke-verified on AI-920)

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Install dependencies
        run: npm ci

      - name: Set package name/version/repo
        working-directory: ./packages/openbridge-webcomponents
        run: |
          npm pkg set name="@blue-water-autonomy/openbridge-webcomponents"
          npm pkg set repository.url="git+https://github.com/Blue-Water-Autonomy/openbridge-webcomponents.git"
          base=$(node -p "require('./package.json').version.split('-')[0]")
          version="${base}-bwa.${GITHUB_RUN_NUMBER}"
          npm pkg set version="$version"

      - name: Publish to GAR
        working-directory: ./packages/openbridge-webcomponents
        run: |
          token="$(gcloud auth print-access-token)"
          {
            echo "@blue-water-autonomy:registry=https://us-central1-npm.pkg.dev/blue-water-autonomy-operations/bwa-npm/"
            echo "//us-central1-npm.pkg.dev/blue-water-autonomy-operations/bwa-npm/:_authToken=${token}"
            echo "//us-central1-npm.pkg.dev/blue-water-autonomy-operations/bwa-npm/:always-auth=true"
          } > .npmrc
          npm publish --tag latest
```

The "Set package name/version/repo" step must be preserved **verbatim** from the current file (byte-for-byte: name, repository.url, base/version stamping).

## Test Strategy

No unit/integration test harness applies to a CI publish workflow. Mechanical validation:

- **YAML validity:** parse `.github/workflows/release.yml` with a YAML parser (e.g. `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/release.yml'))"`) — must succeed.
- **Static assertions** (grep/inspection):
  - `name: Publish to GAR` present.
  - `on:` has `push.tags` (glob **`bwa-v*`**, not bare `*`) and `workflow_dispatch`; has **no** `push.branches`.
  - `permissions:` has `id-token: write` and `contents: read`; has **no** `packages: write`.
  - `setup-node` step has **no** `registry-url` and **no** `scope`.
  - `google-github-actions/auth@v2` present with `workload_identity_provider:` set to the AI-920 provider string; **no** `service_account:` anywhere in the file.
  - `google-github-actions/setup-gcloud@v2` present.
  - `.npmrc` written via an `echo`/brace block, **not** a heredoc (`<<`); contains the three GAR lines (registry, `_authToken`, `always-auth`).
  - The "Set package name/version/repo" run block is unchanged from the fork point (`git show dd46479d:.github/workflows/release.yml` diff shows that block untouched).
  - `npm ci` retained; `npm publish --tag latest` retained.
  - No occurrence of `npm.pkg.github.com` or `secrets.GITHUB_TOKEN` / `NODE_AUTH_TOKEN` remains in the file.
- **Runtime verification is out of band:** an actual publish (tag push) and `gcloud artifacts versions list ...` are listed in the ticket's "Done when" but require pushing a tag to the real repo and live GCP — performed by the maintainer after merge, not in this worktree.

## Acceptance Criteria

| # | Criterion | Source |
|---|-----------|--------|
| 1 | `release.yml` modified per above and merged to `blue-water-autonomy` (the actual publish branch). | Ticket |
| 2 | Open Problem 1 (version `2.0.0` vs `0.0.17`) is explicitly decided and coordinated with AI-918; the resolved published version is documented in the PR. | Ticket |
| 3 | Open Problem 2 (WIF condition allows the chosen trigger ref) is confirmed before the first real publish. | Ticket |
| 4 | A publish from the chosen trigger pushes a new `@blue-water-autonomy/openbridge-webcomponents` version to GAR via direct WIF. | Ticket |
| 5 | Version listed in GAR (`gcloud artifacts versions list --repository=bwa-npm --location=us-central1 --project=blue-water-autonomy-operations --package=openbridge-webcomponents`). | Ticket |
| 6 | No `service_account:` in the workflow; no SA key files anywhere. | Ticket |
| 7 | Power-module's e2e (sibling AI-918) resolves and installs the package from GAR at the agreed version. | Ticket |

**Verifiable within this worktree:** #1 (file modified per spec; merge-to-branch handled by the PR delivery), #2 (decision recorded here and to be in the PR body), #3 (CEL verified from AI-920 Done state; trigger chosen to satisfy it), #6 (static check of the file + repo).

**Out-of-band (post-merge, maintainer/live-infra):** #4, #5, #7 — require pushing a tag to the real repository and live GCP/power-module runs; not executable from the worktree. The PR must call these out as post-merge verification.
