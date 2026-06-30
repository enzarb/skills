---
name: build-push-image
description: >
  Build and push a container image using the pre-configured BuildKit remote
  builder. Activate when the user asks to build, push, release, or deploy a
  container image (including when deploy implies a build). Also activate
  proactively after commits when a deploy is requested.
---

# build-push-image

The workspace has a BuildKit daemon running as a sidecar (`moby/buildkit:rootless`
on `tcp://localhost:1234`). On first boot, `project-agent` registers it as the
default `docker buildx` builder, so `docker buildx build` routes there automatically.

## Standard build + push

```bash
TAG=$(git rev-parse --short HEAD)

docker buildx build \
  --file deploy/Dockerfile \
  --tag "$REGISTRY/<image-name>:${TAG}" \
  --tag "$REGISTRY/<image-name>:latest" \
  --push \
  .
```

**`$REGISTRY`** is pre-set to `registry.enzarb.dev/{org}/{project}` — use it as
a prefix: `$REGISTRY/myimage:tag`, never `$REGISTRY:tag` alone.

## Rules

- **Always `--push`** — remote builders cannot load images locally; `--load` will fail.
- **Push credentials are automatic — do not wire them up.** `docker buildx` reads
  `$DOCKER_CONFIG`, resolves credentials via the `docker-credential-k8s-sa` helper,
  and forwards the auth token to buildkitd inline in the build request. No `--secret`,
  `docker login`, or extra flags are needed.
- **Never override `$DOCKER_CONFIG`** or pass `--config`. The credential helper is
  pre-wired via `/etc/enzarb/docker/config.json`. Overriding it loses the credentials
  and push will 401.
- **No `--builder` flag needed** — the `enzarb` builder is set as the default on
  pod start.

## Credential flow (for debugging)

Push auth is handled transparently:

1. `docker-credential-k8s-sa` reads the projected ServiceAccount token from
   `/var/run/secrets/enzarb/registry/token` (1h TTL, auto-rotated).
2. The token is presented to Zot as `username=sa-token, password=<token>`.
3. Zot validates it via `authd` (Kubernetes TokenReview).
4. The workspace SA token is only authorized for the `{org}/{project}` path prefix.

## First-build note

The first build after pod start may take 5–15 minutes as BuildKit initializes its
rootless worker and pulls base layers. Subsequent builds hit the layer cache and
are fast.
