# workflows

Shared GitHub Actions workflows for every project that follows the [engineering standards](https://github.com/jakefrenzel/engineering-standards) ([CI standard](https://github.com/jakefrenzel/engineering-standards/blob/main/standards/ci.md)).

Projects call these workflows instead of keeping their own copies, so a fix is made once and reaches every repo.

This repo is public because a private repo's workflows can only be called by repos in the same account. The workflows hold no secrets.

## Workflows

| Workflow | Checks | Inputs |
|---|---|---|
| [`pr-title.yml`](.github/workflows/pr-title.yml) | The PR title follows Conventional Commits. Skipped for Dependabot | None |
| [`gitleaks.yml`](.github/workflows/gitleaks.yml) | No secrets anywhere in the git history | None |
| [`go.yml`](.github/workflows/go.yml) | Formatting (`goimports`), `golangci-lint`, `sqlc diff` (if there's a `sqlc.yaml`), `govulncheck`, and `go test -race` | `working-directory` (default `api`) |
| [`web.yml`](.github/workflows/web.yml) | Prettier, ESLint, TypeScript types, and Vitest | `working-directory` (default `apps/web`) |

They run the same commands as `make lint` and `make test` in the [Makefile template](https://github.com/jakefrenzel/engineering-standards/blob/main/templates/Makefile).

**What the calling repo needs:**

- **Go:** `goimports`, `golangci-lint`, `govulncheck`, and (if used) `sqlc` as `tool` lines in `go.mod` (`go get -tool <module>`). Go's version comes from `go.mod`.
- **Web:** a `.node-version` file and `packageManager` in the root `package.json`, with `prettier`, `eslint`, `typescript`, and `vitest` installed.
- **gitleaks:** findings that aren't real secrets go in a `.gitleaksignore` at the repo root.

## Using them

Add one short caller file per workflow to the project's `.github/workflows/`. The caller decides **when** to run; the shared workflow decides **what** runs.

Always call by **major tag** (`@v1`), never `@main`.

### `.github/workflows/pr-title.yml`

```yaml
name: pr-title

on:
  pull_request_target:
    types: [opened, edited, synchronize, reopened]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  pull-requests: read

jobs:
  pr-title:
    uses: atlasopsec/workflows/.github/workflows/pr-title.yml@v1
```

### `.github/workflows/gitleaks.yml`

```yaml
name: gitleaks

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  gitleaks:
    uses: atlasopsec/workflows/.github/workflows/gitleaks.yml@v1
```

### `.github/workflows/go.yml`

```yaml
name: go

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    paths: ["api/**", ".github/workflows/go.yml"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  go:
    if: github.event.pull_request.draft == false
    uses: atlasopsec/workflows/.github/workflows/go.yml@v1
    # with:
    #   working-directory: .   # for a Go-only repo with go.mod at the root
```

### `.github/workflows/web.yml`

```yaml
name: web

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    paths: ["apps/web/**", "packages/**", "package.json", "pnpm-lock.yaml", ".node-version", ".github/workflows/web.yml"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  web:
    if: github.event.pull_request.draft == false
    uses: atlasopsec/workflows/.github/workflows/web.yml@v1
```

### Notes for callers

- **Permissions:** a shared workflow can't have more permissions than its caller grants, so keep the `permissions:` block in each caller.
- **Concurrency goes in the caller only.** Inside a shared workflow, `github.workflow` is the caller's name, so a `concurrency` group there would match the caller's own group and cancel itself.
- **Path filters:** a workflow skipped by `paths` doesn't report a status. That's fine while branch protection isn't enforced; if a check is ever made required, drop its path filter.
- **End-to-end tests** (Playwright) depend on each app's setup, so they stay in the project's own workflow for now.

## Rules for this repo

- Every workflow follows the [CI standard](https://github.com/jakefrenzel/engineering-standards/blob/main/standards/ci.md): least-privilege `permissions:`, `ubuntu-latest`, `timeout-minutes` on every job, and `persist-credentials: false` on checkout.
- GitHub's own actions (`actions/*`) use a major tag. **Every other action is pinned to a full commit SHA**, with the version in a comment.
- **No secrets** in this repo. If a workflow ever needs one, it takes it as a `secrets:` input from the caller.
- Changes go through pull requests. This repo's own CI ([`ci.yml`](.github/workflows/ci.yml), [`ci-pr-title.yml`](.github/workflows/ci-pr-title.yml)) runs `actionlint` and calls its own workflows by local path, so a PR is checked with the versions it changes.

## Releasing

Releases use [semantic versions](https://semver.org). A **major tag** (`v1`) always points at the latest release in that major version, and that's what projects call.

After a PR merges:

```bash
git checkout main && git pull
git tag v1.0.0              # the new version
git tag -f v1               # move the major tag to it
git push origin v1.0.0
git push -f origin v1       # only the major tag is ever force-pushed
```

- **Patch** (`v1.0.1`): a fix that changes nothing for callers.
- **Minor** (`v1.1.0`): something new that callers can ignore, like a new optional input or a new workflow.
- **Major** (`v2.0.0`): anything that could break a caller, like a removed input, a new required tool, or a stricter check. Create a new `v2` tag and leave `v1` where it is. Each project moves to `v2` in its own PR.
