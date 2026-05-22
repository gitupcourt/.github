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
2. **Actions settings** on this repo: *Settings → Actions → General → Access* → "Accessible from repositories in the 'gitupcourt' organization." Required because the repo is private and reusable workflows in private repos default to disallowed cross-repo access.

## Why this exists

Without this, every app session re-derives the same CI pipeline from scratch and the same conversation gets repeated. Centralizing makes the pattern obvious, audit-able, and tunable in one place. Bumping a build cache, adding image scanning, swapping the registry — all change here, every app benefits.
