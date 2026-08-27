# FORK.md — RA-Trio / posthog

## What this is

This repository is RA-Trio's fork of [posthog-foss](https://github.com/PostHog/posthog-foss), the
public/FOSS mirror of PostHog's main product repo. It exists so that, if and when we ever need to
carry a patch against PostHog, we have our own repo and our own image build pipeline to do it from
— rather than patching a vendored copy with no path to a rebuilt image.

## Zero-delta policy

Until noted otherwise in the delta list below, this fork carries **no code changes** against
upstream `posthog-foss`. It is periodically synced from upstream and is not meant to accumulate
unreviewed drift. Any change made here is:

- tracked explicitly in the delta list below,
- kept as small and as isolated from upstream files as practical (new files over edits to existing
  ones, where the goal allows it), and
- expected to be reviewed against upstream on every sync, since a future upstream change could
  conflict with or obsolete it.

If a real product patch ever needs to land here, this is the doc to update.

## Pinned-deploy convention

Our local stack (`posthog-stack`, compose project `posthog`) does not float on `:master` or
`:latest` for the images that matter most. The main app image is pinned via env vars in
`posthog-stack/.env`:

```
REGISTRY_URL=posthog/posthog
POSTHOG_APP_TAG=<pinned tag/sha>
POSTHOG_NODE_TAG=<pinned tag/sha>
```

`docker-compose.yml` reads these instead of hardcoding a tag, so bumping the pin is a one-line
`.env` edit, not a compose-file edit. The repo clone used to build/deploy the stack
(`posthog-stack/posthog`, this checkout) is itself pinned to the exact upstream commit the running
images were built from — today that's `339c98d8cbbd274493a6db0d9bb72b477facc404`. The intent of
this CI pipeline is to extend that same "pin, don't float" discipline to fork-built images once we
have any.

## Delta list (current)

This is the complete list of what this fork carries beyond a zero-delta sync from `posthog-foss`:

1. `.github/workflows/ra-trio-build-image.yml` — builds and pushes the main PostHog image to GHCR
   (`ghcr.io/ra-trio/posthog`) on manual dispatch or on a `ra-trio-*` tag push. See below.
2. `FORK.md` — this file.

Nothing else. No application code, no Dockerfile, no compose file in this repo has been modified.

## The build pipeline

`.github/workflows/ra-trio-build-image.yml` builds upstream's existing production Dockerfile
(root `./Dockerfile` — the same one PostHog itself uses for self-hosted and, re-tagged, for
PostHog Cloud) completely unmodified. It does not reuse upstream's private composite actions under
`.github/actions/` (`build-n-cache-image`, `docker-meta`) because those assume infrastructure this
fork doesn't have — a paid Depot builder, an AWS role for ECR, DockerHub credentials — so instead it
calls the same underlying `docker/*` actions directly (`setup-qemu-action`, `setup-buildx-action`,
`login-action`, `metadata-action`, `build-push-action`), pinned to the same commit SHAs already
vetted elsewhere in this repo's `.github/actions/*` where they overlap.

Key facts:

- **Triggers**: `workflow_dispatch` (manual) and `push` on tags matching `ra-trio-*` only. It does
  **not** trigger on push to `master` — an upstream sync landing on `master` must never kick off a
  build here.
- **Auth**: standard `GITHUB_TOKEN` with the workflow-level `packages: write` permission — no PAT,
  no org secret, nothing upstream-specific.
- **Target**: `ghcr.io/ra-trio/posthog`, tagged with the long commit SHA, the raw `github.sha`, the
  git tag name (when triggered by a `ra-trio-*` tag push), and an optional operator-supplied tag on
  manual dispatch.
- **Multi-arch**: `linux/amd64` + `linux/arm64` via buildx + QEMU emulation, in one build so both
  land under one manifest list.
- **Caching**: GitHub Actions cache backend (`cache-from`/`cache-to: type=gha`, `mode=max`) — no
  external cache registry needed.
- **Build args**: `COMMIT_HASH` (mirrors upstream's own `build-n-cache-image` action) plus the
  `GITHUB_*` args the Dockerfile's sourcemap-upload stage reads for release metadata. That stage
  also expects a `posthog_upload_sourcemaps_cli_api_key` secret, which this fork does not have and
  is not adding; the Dockerfile already handles that gracefully (its own fallback logic retains
  `.map` files in the image and logs a warning instead of failing the build).

**Status: unverified.** No build has been run yet — this task explicitly excluded doing a local
build (hours of compute for a full multi-arch PostHog image). The first real run will be a manual
`workflow_dispatch` after this PR merges, and that is the point at which the pipeline gets proven
end-to-end (auth, multi-arch, cache, and that the pushed image actually boots).

## Switching the local stack to fork-built images

### The main app image (in scope for this pipeline)

`posthog-stack/docker-compose.yml` has three services reading `$REGISTRY_URL:$POSTHOG_APP_TAG` —
`worker`, `web`, and `asyncmigrationscheck`. That is exactly the image this pipeline builds. Once a
real build has been pushed, point the stack at it with an `.env` edit:

```
REGISTRY_URL=ghcr.io/ra-trio/posthog
POSTHOG_APP_TAG=<sha or ra-trio-* tag from the build you want>
```

(GHCR images are public-pull by default for a public repo; if the package is ever set private,
`docker login ghcr.io` on the deploy host first.)

**Gotcha:** don't stop there — `REGISTRY_URL` is shared. `docker-compose.yml` also derives the
Node-only image from it as `${REGISTRY_URL}-node:${POSTHOG_NODE_TAG}`, used by six other services
(`plugins`, `ingestion-general`, `ingestion-sessionreplay`, `recording-api`, `ingestion-logs`,
`ingestion-traces` — all built from `Dockerfile.node`, not the root `Dockerfile`). This pipeline
does **not** build that image. Blanket-overriding `REGISTRY_URL` redirects those six services to
`ghcr.io/ra-trio/posthog-node`, which won't exist until a second workflow is added for
`Dockerfile.node`. Until then, either:

- add a `docker-compose.override.yml` that overrides `image:` on just `worker`, `web`, and
  `asyncmigrationscheck` to the fork-built image directly (leaving `REGISTRY_URL` /
  `POSTHOG_NODE_TAG` untouched for the node services), or
- add the `Dockerfile.node` build job first (tracked as a follow-up, not part of this delta) and
  then the blanket `REGISTRY_URL` swap is safe.

### The Rust microservices (out of scope for this pipeline — pinning guidance only)

These are not built by this workflow. They currently float on upstream's `:master` tag, pulled
straight from upstream GHCR (`ghcr.io/posthog/posthog/...`), per
`posthog-stack/docker-compose.base.yml`. Per this fork's pinned-deploy convention (above), floating
on `:master` for anything in the deploy path is the thing we're trying to move away from — these
are the remaining exception. Enumerated by compose service name, backing image, and current tag:

| Compose service(s)                          | Image                                          | Current tag |
|----------------------------------------------|-------------------------------------------------|-------------|
| `capture`, `replay-capture`, `capture-ai`     | `ghcr.io/posthog/posthog/capture`                | `master`    |
| `capture-logs`                                | `ghcr.io/posthog/posthog/capture-logs`           | `master`    |
| `property-defs-rs`                            | `ghcr.io/posthog/posthog/property-defs-rs`       | `master`    |
| `feature-flags`                               | `ghcr.io/posthog/posthog/feature-flags`          | `master`    |
| `usage-ingestion`                             | `ghcr.io/posthog/posthog/usage-ingestion`        | `master`    |
| `personhog-replica`                           | `ghcr.io/posthog/posthog/personhog-replica`      | `master`    |
| `personhog-router`                            | `ghcr.io/posthog/posthog/personhog-router`       | `master`    |
| `hypercache-server`                           | `ghcr.io/posthog/posthog/hypercache-server`      | `master`    |
| `livestream`                                  | `ghcr.io/posthog/posthog/livestream`             | `master`    |

(`capture`, `replay-capture`, and `capture-ai` are three compose services running the same
`capture` binary/image in different modes — one image, three services. Each of these compose
service entries also already carries its own `build: {context: rust/, args: {BIN: ...}}` block,
i.e. `docker compose build <service>` can build it locally from this repo's `rust/` tree today;
that's a local escape hatch, not a substitute for a pinned, CI-published tag.)

Until this fork builds and publishes its own Rust-service images, the pragmatic pinning move is to
replace `:master` with a specific upstream digest (`ghcr.io/posthog/posthog/<name>@sha256:...`)
pulled at a known-good point, so at least the local stack stops silently moving underneath itself
on every upstream push to `master` — not because we own the build, but because we've stopped
floating on a tag we don't control. Building these from the fork is a natural follow-up to this
pipeline (extend `.github/rust-images.yml`'s existing per-image list, which already documents
`dockerfile`/`project` per Rust image, into a fork workflow analogous to this one) but is
deliberately out of scope here: the ask for this delta was the main app image only.
