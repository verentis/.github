# Repository structure policy

Verentis uses a hybrid repository model. A Git repository is an ownership, access, and
release-coordination boundary; package manifests identify whether an installable artifact is an application
or execution engine.

## Repository types

### Dedicated product repositories

Use a repository named for the product or customer domain when any of these conditions applies:

- the code may be handed over to a customer or partner;
- it needs distinct access, licensing, legal ownership, or security controls;
- it has a substantial independent roadmap or contributor community;
- its build or deployment lifecycle is materially different from related packages;
- repository-level issues and releases are part of the product identity.

Dedicated repositories do not use `-application` or `-engine` suffixes. Examples are `graphite`,
`wholesale-seeds`, and `canvas`.

### Application collections

Small applications may share a repository when they have the same source visibility and maintainer group,
use compatible build and CI tooling, and remain independently versioned and publishable.

- `apps` contains reusable applications intended to become public after readiness review.
- `internal-apps` contains protected and operational applications that remain internal.

Each package lives under `packages/<package-name>/` and owns its manifest, changelog, marketplace assets,
deployment configuration, and release version.

### Runtime repositories

Execution ecosystems use `<runtime>-runtime` names, such as `python-runtime`. A runtime repository should
co-locate the engine, its SDK, compatibility tests, and runtime-specific examples when they share
maintainers and compatibility releases.

Runtime package roots remain independent:

```text
<runtime>-runtime/
  engine/
  sdk/
  examples/
  tests/
  tools/
```

### Templates and examples

`templates` contains independently versioned application, engine, and engine-SDK templates. The `examples`
repository is reserved for cross-runtime or platform-wide sample workspaces; runtime-specific examples
belong in the corresponding runtime repository.

## Visibility policy

Reusable, non-customer-specific applications, engines, SDKs, and templates are public by default only
after an explicit readiness review. Public and non-public packages must not share a repository.

The readiness review covers:

- secrets and sensitive history;
- dependency and source licensing;
- security boundaries and disclosure policy;
- documentation, contribution guidance, and architecture;
- branding and customer-specific content;
- supported versions and maintenance expectations.

## Collection split-out criteria

Move a package to a dedicated repository when its visibility, ownership, handover requirements, roadmap,
or build lifecycle diverges from the collection. Preserve directory history during extraction and keep the
marketplace package identity stable unless a separate package migration is approved.

## Naming rules

- Name product repositories for the product or domain.
- Name collection repositories for their portfolio purpose.
- Name runtime repositories `<runtime>-runtime`.
- Do not encode package type in every repository name.
- Continue using `.app.yaml` and `.engine.yaml` manifest suffixes because artifact type is operationally
  meaningful at the package level.

## Migration rules

- Preserve Git history when combining, renaming, or extracting repositories.
- Separate repository renames from marketplace package-ID changes.
- Make automation consume the portfolio catalog rather than infer policy from names.
- Validate and publish each package from its own package root.
- Keep redirects and archived repositories only until downstream consumers migrate.

## Collection CI

Collection repositories should call the organization workflows in `.github/workflows`:

- `discover-verentis-packages.yml` builds a changed-package matrix from application and engine manifests;
- `validate-verentis-package.yml` validates one independently packaged root;
- `publish-verentis-package.yml` validates, versions, packs, and publishes one package.
- `publish-verentis-package-sprint.yml` uses a protected `sprint` environment, GitVersion,
  publisher GitHub federation, and a developer signing key to publish each production-overlay
  package to Sprint. Its callers must pass the repository-accessible organization secrets
  `SPRINT_API_URL` and `PUBLISHER_SIGNING_KEY`, request `id-token: write`, and register their
  repository/environment as a federated credential on the Sprint publisher before publishing.

Repository-level shared paths can deliberately invalidate every package. Package-specific changes should
validate and release only the owning package root.
