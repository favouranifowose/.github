# Shared delivery workflows

Reusable CI/CD and organization defaults for `favouranifowose`. App repositories use Bun, SvelteKit on Cloudflare Workers, Convex backends, and Pulumi TypeScript infrastructure. Pulumi manages Cloudflare/GitHub infrastructure (domains, scoped tokens, rulesets); Wrangler uploads Worker code. Convex deployment is outside these workflows.

| Workflow | Behavior | GitHub token permissions |
| --- | --- | --- |
| `ci.yml` | Frozen install, `check`, `test`, optional `build` | `contents: read` |
| `deploy-worker.yml` | Build, then `versions upload` for preview or `deploy` for production; one sticky PR comment per environment/directory | Deploy: `contents: read`; separate comment job: `pull-requests: write` |
| `pulumi.yml` | Validate `preview`/`up`, frozen install, run Pulumi; preview posts a sticky PR summary | `contents: read`; preview job also gets `pull-requests: write` |
| `lint.yml` | On PRs and pushes to `main`, run actionlint and zizmor (regular persona, fail on high findings) | `contents: read` |
| `release.yml` | On stable `vMAJOR.MINOR.PATCH` tags, create a release with generated notes and advance the major tag | Release job: `contents: write` |

All jobs have timeouts. CI supersedes older runs on the same ref; preview uploads supersede older runs for the same PR/environment/directory. Worker production deployments serialize by environment/directory, and Pulumi operations serialize by stack without canceling an active operation. GitHub concurrency keeps at most one running and one pending run per group, so intermediate queued runs can be superseded.

## Inputs and secrets

### `ci.yml`

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `bun-version` | string | `1.3.11` | Exact Bun version |
| `working-directory` | string | `.` | Package and lockfile directory |
| `run-build` | boolean | `true` | Build after checks and tests |

No secrets required. The package must provide noninteractive `check`, `test`, and (when enabled) `build` scripts and a committed `bun.lock`.

### `deploy-worker.yml`

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `environment` | string | required | Both the GitHub environment and named Wrangler environment |
| `preview` | boolean | `false` | Upload a version without switching active traffic |
| `working-directory` | string | `.` | App and Wrangler configuration directory |
| `bun-version` | string | `1.3.11` | Exact Bun version |

| Secret | Required | Meaning |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes | Scoped token for this account/Worker and the deployment's required bindings |
| `CLOUDFLARE_ACCOUNT_ID` | yes | Account owning the Worker |

Use `@sveltejs/adapter-cloudflare`, commit Wrangler as a dev dependency with the lockfile, and configure `[env.preview]` and `[env.production]` (or the JSONC equivalents). `preview` should identify a separate Worker from production. Enable `workers_dev` and `preview_urls` for preview and ensure required bindings exist before uploading versions. The Wrangler action uses the installed Wrangler version; use Wrangler 3.88.0 or later so `versions upload` returns a structured preview URL. The comment job fails clearly if a Workers preview URL is missing.

Named environments must include their own bindings where Wrangler does not inherit them. Match Worker names, routes and bindings to the infrastructure owned by Pulumi. The SvelteKit build must produce the entry point/assets configured for that environment. A version preview uses existing resources and may write to its bound databases or services; use isolated preview bindings. Environment secrets override explicitly passed secrets of the same name.

### `pulumi.yml`

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `command` | string | required | Exactly `preview` or `up`; other commands fail validation |
| `stack` | string | required | Fully qualified stack, e.g. `favouranifowose/site/prod` |
| `working-directory` | string | `.` | `Pulumi.yaml`, package and lockfile directory |
| `environment` | string | empty | Required for `up`; preview uses GitHub environment `preview` if omitted |

| Secret | Required | Meaning |
| --- | --- | --- |
| `PULUMI_ACCESS_TOKEN` | yes | Pulumi Cloud token for the stack |
| `CLOUDFLARE_API_TOKEN` | yes | Scoped infrastructure provider token |
| `GH_APP_ID` | no | GitHub App ID for the Pulumi GitHub provider |
| `GH_APP_PRIVATE_KEY` | no | App private key; provide both App secrets or neither |

The Pulumi TypeScript project should select Bun in `Pulumi.yaml` (`runtime.name: nodejs`, `runtime.options.packagemanager: bun`). Pin provider packages in `bun.lock`; Pulumi uses the shared default Bun version. The optional App token is restricted to the caller repository with `administration: write` (for repository settings and rulesets), and exposed as `GITHUB_TOKEN` only to the Pulumi step. Install the App on the intended repositories with that permission. Organization-wide GitHub administration requires a separately reviewed workflow with explicit repository and permission scopes. Without App credentials, the GitHub provider receives the job token, which cannot administer repository settings. Pulumi PR comments always use the job token rather than the provider App token.

### `actions/setup-bun`

| Input | Default | Meaning |
| --- | --- | --- |
| `bun-version` | `1.3.11` | Exact Bun version |
| `registry-token` | empty | Optional token with GitHub Packages `read:packages` and access to the package |

The composite action installs Bun without an executable cache. If a registry token is supplied, it appends the `@favouranifowose` GitHub Packages registry and authentication to the runner user's `.npmrc` with a restrictive creation mask. Use GitHub-hosted ephemeral runners; do not upload that file as an artifact. The reusable workflows do not request a package token; callers needing private packages can use the composite directly in their own jobs (with `packages: read` if using the job token).

```yaml
- uses: favouranifowose/.github/actions/setup-bun@v1
  with:
    registry-token: ${{ secrets.GITHUB_TOKEN }}
```

## App caller example

Save as `.github/workflows/delivery.yml` in a SvelteKit app with its Pulumi project in `infra/`. Replace `site` in the stack name. CI runs for forks; credential-bearing jobs only run for same-repository PRs or pushes to `main`. Never switch this to `pull_request_target` or run untrusted fork code with deployment credentials. Same-repository PRs are trusted code and must be reviewed accordingly.

Provision the `preview` and `production` GitHub environments with Pulumi before using this caller. **Production must require Favour's approval, prevent self-review, and allow deployments only from `main`.** Merely naming an environment does not create an approval rule. Scope preview credentials to preview resources; configure infrastructure credentials separately where their required scope differs from Worker credentials. The example uses the same secret names in each selected environment. Supply required secrets through organization/repository secrets or explicit mappings; use environment-scoped values for deployment overrides.

```yaml
name: Delivery
on:
  pull_request:
  push:
    branches: [main]

permissions: {}

jobs:
  ci:
    permissions:
      contents: read
    uses: favouranifowose/.github/.github/workflows/ci.yml@v1

  infra-preview:
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    needs: ci
    permissions:
      contents: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/pulumi.yml@v1
    with:
      command: preview
      stack: favouranifowose/site/prod
      working-directory: infra
      environment: preview
    secrets:
      PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      GH_APP_ID: ${{ secrets.GH_APP_ID }}
      GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}

  worker-preview:
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    needs: ci
    permissions:
      contents: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/deploy-worker.yml@v1
    with:
      environment: preview
      preview: true
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

  infra-up:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: ci
    permissions:
      contents: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/pulumi.yml@v1
    with:
      command: up
      stack: favouranifowose/site/prod
      working-directory: infra
      environment: production
    secrets:
      PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      GH_APP_ID: ${{ secrets.GH_APP_ID }}
      GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}

  worker-production:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: infra-up
    permissions:
      contents: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/deploy-worker.yml@v1
    with:
      environment: production
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Only CI applies production changes; do not run `pulumi up` locally against production. This example runs infrastructure before Worker production deployment. Both jobs reference the protected production environment and can require separate approvals. An existing preview stack can instead be selected for `infra-preview`; a preview of the production stack only computes a plan and must have sufficient read access to production resources.

## Versioning and maintenance

Callers pin `@v1`. Publish compatible releases with immutable `v1.x.y` tags, then the release workflow creates generated release notes and moves the lightweight `v1` tag to that commit. A breaking interface change requires a new major. The first reviewed release must be tagged before callers or the reusable workflows can resolve the shared `setup-bun@v1` action. No release tags are created as part of opening this repository's implementation PR.

Protect release tags and restrict who may publish them; allow the release job to update the moving major tag. Release tags should point to reviewed commits on `main`, and older release tags should not be pushed after newer ones because each release advances its major alias. Third-party actions are pinned to full commit SHAs with version comments. Renovate groups weekly GitHub Actions updates and preserves these digests; first-party references deliberately follow `@v1`. Enable the Renovate GitHub App for this repository to activate updates.

To check changes locally:

```sh
brew install actionlint zizmor
actionlint
zizmor .
```

The CI security check uses zizmor's regular persona (pedantic off) and fails on high-severity findings. Local `zizmor .` checks all severities by default. No Cloudflare or Pulumi credentials are needed to lint this repository. Live uploads, approval gates, Pulumi operations and releases require configured caller environments and are not exercised by these static checks.
