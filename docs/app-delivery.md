# App delivery (trunk based)

App, engine and package repositories deliver the same way as the platform: short-lived feature branches, pull
requests into `main`, and a staged production rollout on merge. Nothing is rebuilt between stages.

| Event | What happens | Where |
|---|---|---|
| Pull request opened / updated (same repository, not draft) | Changed hosted apps are built and deployed as `<name>-pr-<n>.<apps domain>`; changed packages are published as `<next version>-pr.<n>.<run>.<attempt>`, pointing at the preview host | `feature` environment (internal, behind Cloudflare Access) |
| Pull request closed or merged | The preview copies are removed (`verentis.io/pr=<n>` label). An `expires-at` annotation lets the platform reaper remove leftovers | `feature` |
| Push to `main` (the merge) | Each changed hosted app: build once → `feature` (stable host) → first production region + observation window → remaining regions one at a time. Then each changed package: `feature` registry → global production registry, which distributes it to the regions | `feature`, `production-eus`, `production-nz`, `production-za`, `production` |

A failed stage stops every later stage. Healthy regions are never rolled back automatically; re-run the failed job
after fixing, or revert the commit on `main` (which ships the previous code as a new version).

## Repository wiring

`.github/workflows/pull-request.yml`

```yaml
name: Pull request
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, closed]
permissions:
  contents: read
  id-token: write
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: ${{ github.event.action != 'closed' }}
jobs:
  delivery:
    uses: verentis/.github/.github/workflows/app-pull-request.yml@main
    with:
      build-target: root          # optional: package asset build before packing
    secrets: inherit
```

Also add `GitVersion.yml` at the repository root (copy it from any app repository).

`.github/workflows/release.yml`

```yaml
name: Release
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      everything:
        description: Ship every package and hosted app
        type: boolean
        default: false
permissions:
  contents: write  # the release tags its commit v<version> (GitVersion)
  id-token: write
concurrency:
  group: release
  cancel-in-progress: false
jobs:
  delivery:
    uses: verentis/.github/.github/workflows/app-release.yml@main
    with:
      everything: ${{ inputs.everything || false }}
    secrets: inherit
```

Validation (`validate-verentis-package.yml`) keeps running on every pull request and validates every
`environments/*.yaml` overlay (and the bare manifest when it has no non-local overlays).

To gate shipping on repository checks, make the checks workflow callable (`on: workflow_call`) and have the
`delivery` job need it:

```yaml
jobs:
  checks:
    uses: ./.github/workflows/checks.yml
  delivery:
    needs: checks
    uses: verentis/.github/.github/workflows/app-release.yml@main
    secrets: inherit
```

## Hosted apps: `.github/verentis-delivery.json`

Packages are discovered by manifest suffix. A package that also runs as a workload in Verentis clusters is declared
here; packages without an entry are package-only.

```json
{
  "hosted": [
    {
      "name": "graphite",
      "package-root": ".",
      "image": "verentis/graphite",
      "context": ".",
      "dockerfile": "Dockerfile",
      "manifests": "k8s/deployment.yaml",
      "paths": ["app/", "server/", "Dockerfile", "k8s/", "package-lock.json"],
      "health": [{ "path": "/" }]
    }
  ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Workload and host prefix: `<name>.<apps domain>`; previews `<name>-pr-<n>.<apps domain>` |
| `package-root` | yes | The package this app belongs to; it is republished whenever the app ships |
| `image` | yes | Image repository in the shared registry (`verentis/<...>`) |
| `context`, `dockerfile` | no | Default: `package-root` and `<context>/Dockerfile`. `SOURCE_COMMIT` is passed as a build arg |
| `manifests` | yes, unless `render` | Kubernetes template with `__IMAGE__ __HOST__ __APP__ __NAMESPACE__ __TLS_SECRET__ __APPS_DOMAIN__ __HOST_SUFFIX__ __PLATFORM_ORIGIN__` |
| `render` | no | Command that prints the manifests instead (reads the same values as env vars: `IMAGE HOST APP_NAME NAMESPACE TLS_SECRET APPS_DOMAIN HOST_SUFFIX PLATFORM_ORIGIN SOURCE_COMMIT`) |
| `pre-deploy` | no | Command run against the target cluster before apply (same env vars) |
| `secrets` | no | Names of environment secrets (for example `["CODE_PROOF_KEY"]`) exported to `pre-deploy` and `render` only; a missing one fails the deploy |
| `node` | no | `true` runs `npm ci` before `pre-deploy`/`render` |
| `deployments` | no | Deployment names (without suffix) to wait for. Default `[name]` |
| `health` | no | `[{ "host": "<prefix>", "path": "/_ready" }]`; `host` defaults to `name`. Checked against the target's ingress IP, never through the edge |
| `paths` | no | Path prefixes that trigger a ship. Default `[package-root]` |
| `feature-check` | no | Command run after the feature deploy with `APP_URL`, `PLATFORM_ORIGIN`, `ACCESS_CF_CLIENT_ID/SECRET` |

Per-package publish settings live next to `hosted` and override the workflow's `build-target` input:

```json
{
  "packages": {
    "app": { "build-target": "" },
    "pack-template": { "build-target": "@verentis/template-canvas-component" },
    "packages/load-node": { "publish": false },
    "packages/security-console": { "visibility": "Private" }
  }
}
```

An entry only overrides the keys it sets (`build-target` falls back to the workflow input).
`publish: false` keeps a package (test fixtures, load generators) out of every registry; `visibility` sets the
marketplace visibility of a newly created package.

In a pull request, a hosted app also ships when only its package changed, so the preview package never points at a
preview host that does not exist.

Rules enforced at deploy time: the image is pinned by digest; every object is namespaced to the target's apps
namespace (nothing cluster-scoped) and is one of Deployment, Service, Ingress, ConfigMap, Secret,
PodDisruptionBudget, HorizontalPodAutoscaler, NetworkPolicy or ServiceAccount (what preview removal deletes);
placeholders must all resolve. Production regions roll out strictly in `production-targets` order (up to four). Ingresses use the platform's wildcard certificate
(`__TLS_SECRET__`), so no per-app certificate is issued. Internal environments add the access-header check.

All workload names must use `__APP__` (or `APP_NAME` / `HOST_SUFFIX` in a renderer), so previews never collide with
the stable copy or each other.

## Overlays

| File | Used for | Host |
|---|---|---|
| `environments/local.yaml` | `verentis dev` / local Aspire | `<name>.apps.localtest.me` |
| `environments/feature.yaml` | Pull request previews and the stable feature copy | `https://${param:app-host}` |
| `environments/production.yaml` | Production, every region | `https://<name>.apps.verentis.io` |

`${param:apps-domain}` is supplied to every package at publish time (use it to point at another repository's app,
for example `https://canvas.${param:apps-domain}`); `${param:app-host}` and `${param:host-suffix}` only to a package
with its own hosted app (CLI `--param`). Production hosts are region-free: the edge routes `<name>.apps.verentis.io`
to the workspace's owning region, so a package is published once to the global registry and there are no
per-region production overlays. Use `placement` in the manifest to restrict regions. Package-only packages without
hosts need no `feature.yaml`; a package that has `production.yaml` must also have `feature.yaml`.

## Versions

Versions come from **GitVersion** (`resolve-version` action), one version per repository and release, the same way
the CLI and SDK are versioned. Each repository has a `GitVersion.yml` at its root (main: empty label, patch
increment). A release on `main` takes the last `v<version>` tag plus one patch and pushes `v<version>` to the commit,
which reserves the version: the next merge counts on from it, and re-running the same commit gets the same version.
Nothing is written back to the manifest; `metadata.version` in the repository is ignored at publish time.

- **Patch:** automatic, one per release.
- **Minor or major:** push a tag to the last released commit (for example `git tag v1.3.0` before the next merge),
  or include `+semver: minor` / `+semver: major` in the merge commit message.
- **Pull request previews:** the next version with a `-pr.<n>.<run>.<attempt>` prerelease, which sorts below the
  release that merges them.

Re-running a release is safe: images are pinned by digest, and publishing a version whose content is identical
to what the registry holds is a no-op (a different package under an existing version still fails).

## Trust model

Pull request workflows run the pull request's own code, including `.github/` and `verentis-delivery.json`, with the
`feature` environment's secrets. Anyone who can push a branch to the repository can therefore read them. Keep
`feature` secrets feature-scoped (its own signing key, an identity limited to AcrPush and the feature apps
namespace, the Access service token) and never reuse production values there. Production environments accept
deployments from `main` only; protect `main` with required reviews so only reviewed code reaches them.

## Enabling delivery

Delivery is inert until the repository variable `APP_DELIVERY_ENABLED` is `true`; until then the workflows only print a
notice, so pushing a repository before its environments exist never creates unprotected environments. The platform's
`bootstrap-app-repos` workflow sets it after creating the environments, identities and variables.

## GitHub environments of an app repository

Created and filled by the platform's repository bootstrap; listed here so a missing value is easy to diagnose.

| Environment | Branches | Variables | Secrets |
|---|---|---|---|
| `feature` | any | `AZURE_CLIENT_ID AZURE_TENANT_ID AZURE_SUBSCRIPTION_ID ACR_NAME ACR_LOGIN_SERVER AKS_RESOURCE_GROUP AKS_CLUSTER_NAME APPS_NAMESPACE APPS_DOMAIN APPS_TLS_SECRET INGRESS_IP PLATFORM_ORIGIN VERENTIS_REGISTRY_URL` | `PUBLISHER_SIGNING_KEY ACCESS_HEADER_VALUE ACCESS_CF_CLIENT_ID ACCESS_CF_CLIENT_SECRET` |
| `production-<region>` | `main` | `AZURE_CLIENT_ID AZURE_TENANT_ID AZURE_SUBSCRIPTION_ID AKS_RESOURCE_GROUP AKS_CLUSTER_NAME APPS_NAMESPACE APPS_DOMAIN APPS_TLS_SECRET INGRESS_IP PLATFORM_ORIGIN` | app-specific secrets listed under a hosted app's `secrets` (for example `CODE_PROOF_KEY`, one per environment) |
| `production` | `main` | `VERENTIS_REGISTRY_URL APPS_DOMAIN` | `PUBLISHER_SIGNING_KEY` |

Each environment federates to its own Azure identity with the least it needs: AcrPush for `feature` builds,
"Azure Kubernetes Service Cluster User Role" on its cluster (to fetch the kubeconfig) and a RoleBinding to the
`verentis-app-deployer` Role in the apps namespace, nothing else. Clusters are Entra-integrated with local accounts
disabled, so the workflows convert the kubeconfig with kubelogin (`azure/use-kubelogin`, pinned). The publisher's GitHub pipeline trust
is bound to this repository and the `feature` / `production` environment.

## Prerequisites

- Verentis CLI with `pack --param` and Cloudflare Access headers (`VERENTIS_CF_ACCESS_CLIENT_ID/SECRET`).
- The platform provisions, per cluster, the apps namespace with its isolation baseline and the `*.apps.<domain>`
  wildcard certificate, and keeps the feature platform environment running.
