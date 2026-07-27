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
| `andrew-hesse/workflows` (this one) | reusable workflows | `uses: andrew-hesse/workflows/.github/workflows/<file>@v1` |
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
    uses: andrew-hesse/workflows/.github/workflows/workflow-lint.yml@v1
```

Inputs: `persona` (`regular`, `pedantic`, `auditor`), `min-severity`.

It runs in annotation mode, not SARIF. The action defaults to uploading SARIF to
the security tab, which requires GitHub Advanced Security and therefore fails on
a private repo.

### `docker-publish.yml`

buildx build, pushed to `ghcr.io`, authenticated by the caller's own
`GITHUB_TOKEN`. No stored registry credential.

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: andrew-hesse/workflows/.github/workflows/docker-publish.yml@v1
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

Callers pin `@v1`. The `v1` tag moves as these workflows change, so a fix reaches
every repo without touching each one. Breaking changes get a new major tag.

[zizmor]: https://docs.zizmor.sh/
