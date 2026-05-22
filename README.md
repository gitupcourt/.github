# gitupcourt org-level shared workflows

Reusable GitHub Actions workflows used by every app in the org. See [`profile/README.md`](profile/README.md) for the org-level profile copy.

## What lives here

| Workflow | Purpose |
|---|---|
| [`build-and-bump.yml`](.github/workflows/build-and-bump.yml) | Build one or more container images, push to GHCR with deterministic tags, then (optionally) open a PR in `gitupcourt/homelab` bumping the matching manifest. The default CI for any app deployed to the homelab cluster. |

## How to consume from an app repo

Add `.github/workflows/ci.yml` to the app:

```yaml
name: CI

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  ship:
    uses: gitupcourt/.github/.github/workflows/build-and-bump.yml@main
    with:
      app_name: my-app
      # JSON array — one entry per image the app produces.
      # `context` is the build context dir; `dockerfile` is optional
      # if you don't use the default ./Dockerfile inside the context.
      images: |
        [
          {"name": "backend",  "context": "backend"},
          {"name": "frontend", "context": "frontend", "dockerfile": "frontend/Dockerfile.prod"}
        ]
      # Optional: folder inside the homelab repo to bump.
      # Omit (or set to "") to skip the manifest PR step.
      manifest_dir: manifests/my-app
    secrets: inherit
```

## One-time org setup

1. **Org secret `HOMELAB_PAT`** — a fine-grained PAT scoped to `gitupcourt/homelab` with `contents:write` and `pull_requests:write`. Used by `build-and-bump` to commit a tag bump and open a PR. Without it, the build/push half still works; the manifest-PR half is skipped with a warning.
2. **This repo is public.** GitHub Free doesn't allow private cross-repo reusable workflow access org-wide. The repo only contains CI YAML and README, no secrets — making it public is the standard practice (the GitHub docs assume it).

## Gotchas / one-time per app

These two are the unobvious things that bit us setting up the first consumer. They live in this README so the next app session doesn't repeat them.

### Caller must grant `permissions:` matching what the reusable requests

GitHub's default workflow permissions are `read`. A reusable workflow cannot escalate above what the caller has granted, so this **must** appear on the caller's job:

```yaml
jobs:
  ship:
    permissions:
      contents: read
      packages: write   # required for GHCR push
    uses: gitupcourt/.github/.github/workflows/build-and-bump.yml@main
```

Without it, the workflow validation fails with `startup_failure` and no jobs spawn — and no error text is surfaced through the REST API. Lost an hour to that one.

### GHCR packages must be linked to the calling repo

When packages are first created by `docker push` from a developer machine (not by Actions), they're owned by the user with no repo link. The workflow's `GITHUB_TOKEN` then gets `403 Forbidden` on push.

Fix (one-time, per package, per app): in the GitHub web UI, go to the package's settings page (e.g. `https://github.com/users/<owner>/packages/container/<name>/settings`) → **Manage Actions access** → **Add Repository** → pick the app's repo with role **Write**. Two clicks per package.

Packages created by CI from the start don't need this — they auto-link to the originating repo on creation. The gotcha is migration only.

## Why this exists

Without this, every app session re-derives the same CI pipeline from scratch and the same conversation gets repeated. Centralizing makes the pattern obvious, audit-able, and tunable in one place. Bumping a build cache, adding image scanning, swapping the registry — all change here, every app benefits.
