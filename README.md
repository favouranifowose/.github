# Shared delivery workflows

Reusable CI/CD and organization defaults for `favouranifowose`. App repositories use Bun, SvelteKit on Cloudflare Workers, Convex backends, and Pulumi TypeScript infrastructure. Pulumi manages Cloudflare/GitHub infrastructure (domains, scoped tokens, rulesets); Wrangler uploads Worker code. Convex deployment is outside these workflows.

Private app repositories use GitHub Free with no environment approval gates. Agents may push directly to `main`; pipelines check the code, apply infrastructure, and deploy Worker code from `main`. PRs are optional and can provide previews before a change lands.

| Workflow | Behavior | GitHub token permissions |
| --- | --- | --- |
| `ci.yml` | Frozen install, `check`, `test`, optional `build` | `contents: read`, `packages: read` |
| `deploy-worker.yml` | Build, then `versions upload` for preview or `deploy` for production; one sticky PR comment per environment/directory | Deploy: `contents: read`, `packages: read`; separate comment job: `pull-requests: write` |
| `pulumi.yml` | Validate `preview`/`up`, frozen install, run Pulumi; preview posts a sticky PR summary | Install jobs: `contents: read`, `packages: read`; preview also gets `pull-requests: write` |
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

No additional secrets required: installs authenticate to GitHub Packages with the job's `github.token`. Grant each caller repository Actions access to the private `@favouranifowose/infra` package in the package's settings; `packages: read` alone does not grant cross-repository access. The package must provide noninteractive `check`, `test`, and (when enabled) `build` scripts and a committed `bun.lock`.

### `deploy-worker.yml`

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `environment` | string | empty | Optional GitHub environment; omit on GitHub Free private repositories |
| `wrangler-environment` | string | empty | Optional Wrangler `--env`; empty uses the top-level Wrangler configuration |
| `preview` | boolean | `false` | Upload a version without switching active traffic |
| `working-directory` | string | `.` | App and Wrangler configuration directory |
| `bun-version` | string | `1.3.11` | Exact Bun version |

| Secret | Required | Meaning |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes | Scoped token for this account/Worker and the deployment's required bindings |
| `CLOUDFLARE_ACCOUNT_ID` | yes | Account owning the Worker |

Use `@sveltejs/adapter-cloudflare` and commit Wrangler as a dev dependency with the lockfile. The GitHub `environment` does not select a Wrangler environment: leave `wrangler-environment` empty for apps with only a top-level `wrangler.jsonc`. Set it explicitly only when the app defines a matching named Wrangler environment. Enable `workers_dev` and `preview_urls` for version previews and ensure required bindings exist before uploading versions. The Wrangler action uses the installed Wrangler version; use Wrangler 3.88.0 or later so `versions upload` returns a structured preview URL. In pinned wrangler-action v4.1.3, `versions upload` exposes its `preview_url` as **`deployment-url`**, which feeds the GitHub environment URL and sticky PR comment; `preview-url` is for the separate `wrangler preview` command. See the pinned [action metadata](https://github.com/cloudflare/wrangler-action/blob/953926a2e2182532811c01a25e53647d93bf07c0/action.yml) and [version upload output handler](https://github.com/cloudflare/wrangler-action/blob/953926a2e2182532811c01a25e53647d93bf07c0/src/commandOutputParsing.ts). The comment job fails clearly if a Workers preview URL is missing.

When used, named Wrangler environments must include their own bindings where Wrangler does not inherit them. Match Worker names, routes and bindings to the infrastructure owned by Pulumi. The SvelteKit build must produce the configured entry point/assets. A version preview uploads without switching active traffic, but uses existing resources and may write to its bound databases or services; select an isolated preview Worker/bindings when that matters. Worker and Pulumi installs use the same GitHub Packages authentication as CI.

Leave `environment` empty for the default delivery model and keep credentials in repository secrets. The workflows select separate jobs with no `environment` key when this input is empty. [GitHub documents environment references by name](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idenvironment), without guaranteeing that an empty name disables the reference; it also [requires a paid plan for environments in private repositories](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments). Explicitly setting a nonempty input opts into a GitHub environment on repositories whose plan supports it; that environment's configured rules and secrets then apply. `wrangler-environment` remains independent.

### `pulumi.yml`

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `command` | string | required | Exactly `preview` or `up`; other commands fail validation |
| `stack` | string | required | Fully qualified stack, e.g. `favouranifowose/site/prod` |
| `working-directory` | string | `.` | `Pulumi.yaml`, package and lockfile directory |
| `environment` | string | empty | Optional GitHub environment for either command; empty uses jobs with no environment reference |

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

The composite action installs Bun without an executable cache. If a registry token is supplied, it appends the `@favouranifowose` GitHub Packages registry and authentication to the runner user's `.npmrc` with a restrictive creation mask. [Bun supports user-level `.npmrc` files and scoped registries](https://bun.sh/docs/pm/npmrc); no additional `bunfig.toml` is needed. All reusable jobs that install dependencies pass `github.token` and grant `packages: read`. Callers must also allow `packages: read` and have Actions access to each private package. Use GitHub-hosted ephemeral runners; do not upload `.npmrc` as an artifact. The composite can also be used directly:

```yaml
- uses: favouranifowose/.github/actions/setup-bun@v1
  with:
    registry-token: ${{ github.token }}
```

## App caller example

Save as `.github/workflows/delivery.yml` in a SvelteKit app with its Pulumi project in `infra/`. Replace `site` in the stack name. Store credentials in repository secrets and grant the repository Actions access to the private infra package. Every push to `main` runs CI, then `pulumi up`, then the Worker deployment; there is no manual approval step.

The PR trigger and preview jobs below are optional: remove them if previews are not needed. CI runs for fork PRs; credential-bearing preview jobs run only for same-repository PRs. Never switch this to `pull_request_target` or run untrusted fork code with deployment credentials. Same-repository PRs execute trusted code. Use separately scoped preview credentials and isolated bindings where needed.

```yaml
name: Delivery
on:
  push:
    branches: [main]
  pull_request: # Optional: remove this trigger and the two preview jobs below.

permissions: {}

jobs:
  ci:
    permissions:
      contents: read
      packages: read
    uses: favouranifowose/.github/.github/workflows/ci.yml@v1

  infra-up:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: ci
    permissions:
      contents: read
      packages: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/pulumi.yml@v1
    with:
      command: up
      stack: favouranifowose/site/prod
      working-directory: infra
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
      packages: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/deploy-worker.yml@v1
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

  # Optional PR previews.
  infra-preview:
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    needs: ci
    permissions:
      contents: read
      packages: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/pulumi.yml@v1
    with:
      command: preview
      stack: favouranifowose/site/prod
      working-directory: infra
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
      packages: read
      pull-requests: write
    uses: favouranifowose/.github/.github/workflows/deploy-worker.yml@v1
    with:
      preview: true
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Only CI applies production changes; do not run `pulumi up` locally against production. This example runs infrastructure before Worker production deployment, without referencing any GitHub environment. An existing preview stack can instead be selected for `infra-preview`; a preview of the production stack only computes a plan and must have sufficient read access to production resources. Callers grant PR write permission because the reusable workflows include preview/comment jobs; the production execution jobs narrow their own job permissions.

## Versioning and maintenance

Callers pin `@v1`. Publish compatible releases with immutable `v1.x.y` tags, then the release workflow creates generated release notes and moves the lightweight `v1` tag to that commit. A breaking interface change requires a new major. The shared `setup-bun@v1` action follows the same supported major.

Protect release tags and restrict who may publish them; allow the release job to update the moving major tag. Release tags should point to reviewed commits on `main`, and older release tags should not be pushed after newer ones because each release advances its major alias. Third-party actions are pinned to full commit SHAs with version comments. Renovate groups weekly GitHub Actions updates and preserves these digests; first-party references deliberately follow `@v1`. Enable the Renovate GitHub App for this repository to activate updates.

To check changes locally:

```sh
brew install actionlint zizmor
actionlint
zizmor .
```

The CI security check uses zizmor's regular persona (pedantic off) and fails on high-severity findings. Local `zizmor .` checks all severities by default. No Cloudflare or Pulumi credentials are needed to lint this repository. Live uploads and Pulumi operations require caller credentials and are not exercised by these static checks.
