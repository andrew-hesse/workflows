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
| `andrew-hesse/workflows` (this one) | reusable workflows | `uses: andrew-hesse/workflows/.github/workflows/<file>@f5f8d60f7bb85aa6486cdfacad9ef96c76751323 # v3.2` |
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
    uses: andrew-hesse/workflows/.github/workflows/workflow-lint.yml@f5f8d60f7bb85aa6486cdfacad9ef96c76751323 # v3.2
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
    uses: andrew-hesse/workflows/.github/workflows/docker-publish.yml@f5f8d60f7bb85aa6486cdfacad9ef96c76751323 # v3.2
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

### `node-ci.yml`

The standard pnpm PR gate: install, then lint, typecheck, test and build.

```yaml
jobs:
  ci:
    permissions:
      contents: read
    uses: andrew-hesse/workflows/.github/workflows/node-ci.yml@f5f8d60f7bb85aa6486cdfacad9ef96c76751323 # v3.2
    with:
      node-version-file: package.json
```

This needs no input per script only because the script vocabulary is
standardised (see `CI.md` in `ahesse/docs`): `lint`, `typecheck`, `test` and
`build` mean the same thing in every repo.

Inputs: `node-version`, `node-version-file`, `test-script` (default `test`),
`run-build`, `timeout-minutes`, `npm-token-op-ref`.

Point `test-script` at a stricter script where the repo has one. A repo whose
gate was a coverage run with enforced floors would otherwise have to re-run its
whole suite in a second job to keep that floor, since a gate may get stronger
but never weaker.

A repo needing setup the vocabulary cannot express (a system binary, a service
container) keeps its own `ci.yml`. It does **not** pass a shell command in as an
input: injecting `run:` content through an input is the template-injection shape
the linter flags.

### `e2e.yml`

The Playwright suite: install, install browsers, run one script, upload the
report.

```yaml
jobs:
  e2e:
    permissions:
      contents: read
    uses: andrew-hesse/workflows/.github/workflows/e2e.yml@f5f8d60f7bb85aa6486cdfacad9ef96c76751323 # v3.2
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
container. What the container buys is a preinstalled browser set, and that
install takes under a minute on a GitHub-hosted VM. What it costs: the image
bakes its own Node, so a repo on a different `engines.node` layers `setup-node`
on top regardless; its tag duplicates the `@playwright/test` version and can
drift from it, and a mismatch makes Playwright re-download browsers at run time;
and its overlayfs file locking is slow enough that wrangler's local D1 loses a
startup race and fails with `SQLITE_BUSY` unless that state sits on tmpfs.

Browsers are not cached: the key has to track the resolved Playwright version
exactly, a stale entry gives a confusing mismatch rather than a clean miss, and
`--with-deps` installs the OS packages either way.

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
2. Tag the new commit. **Bump the major only for a change a caller must react
   to**: a removed or renamed input, a new required input, a narrower default.
   Anything additive gets a point release (`v3.1`).
3. Repoint the usage examples, in this README and in each workflow's header
   comment, at the new tag. They are what a new caller copies, so an example
   naming an older release wires that caller to it. Resolve the commit with
   `git ls-remote <url> 'refs/tags/vX.Y^{}'`; the annotated tag's own hash is not
   it.
4. Renovate raises the pin bump in each consuming repo, since it manages the
   `github-actions` datasource and updates SHA pins with their comments.

Callers therefore stay on a known-good commit until a bump is reviewed, rather
than silently inheriting whatever `v1` points at today.

Step 2 is not cosmetic. Renovate reads `workflow-lint.yml@<sha> # v3.1` as
version `v3.1` at that digest, so the tag decides which lane the bump takes: a
point release is grouped with the non-major updates and auto-merges on the
fortnightly schedule, while a major sits behind manual review and a 14 day
release-age gate. Numbering a purely additive release as a new major puts a
no-op change through the slowest path, and callers drift in the meantime.

[zizmor]: https://docs.zizmor.sh/
