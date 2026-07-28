# workflows

Reusable GitHub Actions workflows shared across the `andrew-hesse` repos.

This repo is **public** because a personal account cannot share a private
reusable workflow with another repository; that setting is organization-only.
Nothing here is a secret: the workflows reference credentials by name and resolve
them from the caller's own `secrets`.

Companion repos, deliberately kept separate because each has a different
consumption mechanism:

| Repo | Holds | Consumed as |
| --- | --- | --- |
| `andrew-hesse/workflows` (this one) | reusable workflows | `uses: andrew-hesse/workflows/.github/workflows/<file>@<commit> # v1` |
| `andrew-hesse/biome-config` | shared Biome config | `github:andrew-hesse/biome-config#<tag>` in `package.json` |
| `andrew-hesse/renovate` | shared Renovate preset | `github>andrew-hesse/renovate` in `renovate.json` |

They cannot be merged into one repo. npm and pnpm cannot install a
**subdirectory** of a git repo, so the Biome package has to sit at a repo root.
Reusable workflows must sit directly in `.github/workflows/`; GitHub does not
support subdirectories there.

## Workflows

### `workflow-lint.yml`

Audits the calling repo's own workflows with [zizmor], which catches Actions
security problems that `actionlint` does not: credential persistence, template
injection, over-broad permissions, unpinned actions.

```yaml
jobs:
  workflow-lint:
    uses: andrew-hesse/workflows/.github/workflows/workflow-lint.yml@0543eb81752c94e8840b7e43484edd20940b35a0 # v1
```

Pin the **commit**, with the tag as a trailing comment. A bare `@v1` fails
zizmor's own `unpinned-uses` audit, so the linter rejects the line that calls it.
Resolve the commit an annotated tag points at with
`git ls-remote <url> 'refs/tags/v1^{}'`; the tag object's own hash is not it.

Inputs: `persona` (`regular`, `pedantic`, `auditor`), `min-severity`.

It runs in annotation mode, not SARIF. The action defaults to uploading SARIF to
the security tab, which requires GitHub Advanced Security and therefore fails on
a private repo.

### Reproducing the gate locally

The action runs zizmor's **online** audits by default, so an offline local run is
weaker than CI and will pass code that CI rejects. On one repo this was the
difference between 0 findings and 18. Always include the token:

```sh
GH_TOKEN=$(gh auth token) uvx zizmor --min-severity low .
```

The online-only audit that matters most is `ref-version-mismatch`: a SHA whose
trailing comment names a tag the SHA does not actually point at (for example
`actions/checkout@9c091bb…` commented `# v7` when that commit is `v7.0.0`). Fix
the comment; never bump the SHA to match the comment.

### `docker-publish.yml`

buildx build, pushed to `ghcr.io`, authenticated by the caller's own
`GITHUB_TOKEN`. No stored registry credential.

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: andrew-hesse/workflows/.github/workflows/docker-publish.yml@0543eb81752c94e8840b7e43484edd20940b35a0 # v1
    with:
      image-name: andrew-hesse/my-app
      title: my-app
      description: What the image is
```

**The caller must grant `packages: write`.** A reusable workflow can only narrow
the caller's token, never widen it, so declaring it here is not enough.

Set `push: false` to build without publishing, which is how a PR validates a
Dockerfile before merge.

The caller keeps its own `on:` triggers, `paths:` filter and `concurrency` group.
Those are per-repo decisions: the `paths:` filter depends on the repo's layout,
and the `concurrency` group is what stops two merges racing to publish `:latest`.

### `e2e.yml`

The Playwright suite: install, install browsers, run one script, upload the
report.

```yaml
jobs:
  e2e:
    permissions:
      contents: read
    uses: andrew-hesse/workflows/.github/workflows/e2e.yml@<commit> # v3.1
    with:
      node-version-file: package.json
```

The caller's `playwright.config.ts` owns how the suite runs. In particular its
`webServer.command` is where an app that needs building, migrating or serving
before the tests does that work, so a repo whose e2e needs `wrangler dev` and a
D1 migration pass needs no extra inputs here.

Inputs: `node-version`, `node-version-file`, `browsers` (default `chromium`,
empty string for all), `test-script` (default `test:e2e`), `timeout-minutes`,
`artifact-name`, `retention-days`, `npm-token-op-ref`.

It runs on `ubuntu-latest`, **not** in the `mcr.microsoft.com/playwright`
container. The container's purpose is skipping the browser install, which is
under a minute on a GitHub-hosted VM. Avoiding it also avoids the container's own
costs: the image bakes its own Node (so a newer `engines.node` needs `setup-node`
layered on anyway), its tag duplicates the `@playwright/test` version, and its
overlayfs is slow enough at file locking to make wrangler's local D1 lose a
startup race and fail with `SQLITE_BUSY`.

Browsers are not cached: the key must track the resolved Playwright version
exactly or produce a confusing mismatch instead of a clean miss, and `--with-deps`
still has to install the OS packages either way.

## Conventions these encode

- Every action is pinned to a **commit SHA** with the version in a trailing
  comment. Tags are mutable, so `@v4` is a supply-chain hole.
- `persist-credentials: false` on every checkout. By default the job token is
  written to `.git/config`, where any later step can read it; none of these jobs
  push.
- `permissions: {}` at workflow level, with each job granting only what it needs.
- An explicit `timeout-minutes`, so a hung job does not run to GitHub's 6 hour
  default.

## Versioning

Callers pin a **commit**, with the tag name as a trailing comment. That is
immutable, and it is what zizmor's `unpinned-uses` audit requires.

The trade-off is deliberate: a fix here does **not** reach callers by itself, and
the `v1` tag must **not** be moved onto a later commit. Moving it would make every
caller's `# v1` comment name a tag its pinned commit no longer belongs to, which
is exactly what zizmor's `ref-version-mismatch` audit fails on.

So releases work like any other dependency:

1. Change a workflow here, merge it.
2. Tag the new commit (`v2`, or `v1.1` for a compatible change).
3. Renovate raises the pin bump in each consuming repo, since it manages the
   `github-actions` datasource and updates SHA pins with their comments.

Callers therefore stay on a known-good commit until a bump is reviewed, rather
than silently inheriting whatever `v1` points at today.

[zizmor]: https://docs.zizmor.sh/
