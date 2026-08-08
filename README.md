# runner-image

`ghcr.io/qtsone/runner` — the Actions Runner Controller (ARC) runner image used by our
self-hosted runner scale sets.

Built from `ghcr.io/actions/actions-runner`, with two things added: a real `ENTRYPOINT`,
and a pre-populated tool cache.

## Why this image exists

**It sets `ENTRYPOINT`.** The upstream image ships `Entrypoint: null` and `Cmd: ["/bin/bash"]`.
It only works because the Helm chart supplies `command: ["/home/runner/run.sh"]`. When a values
override dropped that key, the container ran `/bin/bash`, read EOF on a TTY-less stdin, and
exited 0 with no log output — runner pods churned every few seconds while jobs sat queued
forever. Setting `ENTRYPOINT` makes that class of failure structurally impossible.

**It pre-bakes the toolchain.** ARC runners are ephemeral, so every job on a stock image
re-downloads and re-installs its entire toolchain. Baking the tools into the image converts
that repeated network+install cost into a one-time image pull that stays in the node's
containerd cache.

## What is baked in

| Tool | Version | Where | Found by |
|---|---|---|---|
| Node.js | 24.19.0 | `${RUNNER_TOOL_CACHE}/node/24.19.0/x64` | `actions/setup-node` |
| Python | 3.12.13 | `${RUNNER_TOOL_CACHE}/Python/3.12.13/x64` | `actions/setup-python` |
| uv | 0.11.32 | `${RUNNER_TOOL_CACHE}/uv/0.11.32/x86_64` | `astral-sh/setup-uv` |
| buf | 1.50.0 | `${RUNNER_TOOL_CACHE}/buf/1.50.0/x64` | `bufbuild/buf-setup-action` |
| bun | 1.3.14 | `/home/runner/.bun/bin/bun` | `oven-sh/setup-bun` |
| gh | 2.96.0 | `/usr/local/bin/gh` | on `PATH` |
| kubectl | 1.36.3 | `/usr/local/bin/kubectl` | on `PATH` |
| kustomize | 5.8.1 | `/usr/local/bin/kustomize` | on `PATH` |
| jq | 1.8.2 | `/usr/local/bin/jq` | on `PATH` |
| yq | 4.53.3 | `/usr/local/bin/yq` | on `PATH` |
| semantic-release | 24.2.0 | global in the baked Node | on `PATH` |

Docker CLI and buildx come from the upstream base image and are not re-installed here.
`gh`, `kubectl`, `kustomize`, `jq`, and `yq` (mikefarah's Go implementation) are plain
system CLIs on `PATH` rather than tool-cache entries, so they need no `setup-*` action
and no version pin in the workflow — they are used as-is. When bumping `KUBECTL_VERSION`, keep it within one minor version of the
target cluster's API server (the supported client/server skew range).

`pnpm` is deliberately **not** baked: `pnpm/action-setup` deletes its install directory and
reinstalls from npm on every run, so a pre-installed copy is never used.

The baked Node's `bin` directory is on `PATH`, so `node`, `npm`, and `semantic-release`
work natively without a `setup-node` step — the same as GitHub-hosted runners.
`semantic-release` 24.2.0 is installed globally together with `@semantic-release/changelog`
7.0.0 and `@semantic-release/git` 11.0.1: exactly what `qtsone/actions/release` installs at
run time (its `semantic-version` input default plus its default `extra-plugins`). Keep the
`SEMANTIC_RELEASE_*` args in lockstep with that action when bumping either side.

## Getting a cache hit

A baked tool is only used if the workflow asks for a version that matches it. The rules differ
per action, and two of them need the workflow to pin a version:

- `setup-node` and `setup-python` resolve semver **ranges** against the cache, so
  `node-version: 24` and `python-version: "3.12"` hit the baked copies as-is.
- `buf-setup-action` defaults to `1.50.0`, which is what is baked — a hit without changes.
- `setup-uv` resolves `latest` to a concrete version *before* checking the cache, so the
  baked copy stops being used the day a newer uv ships. Pin `version: 0.11.32` for a
  deterministic hit.
- `setup-bun` refuses to reuse an existing install whenever the requested version is
  `latest`, `canary`, or unset. Pin `bun-version: 1.3.14` or it always re-downloads.

## Tool cache location

`RUNNER_TOOL_CACHE` is set to `/opt/hostedtoolcache` in the image, and this is load-bearing.
The runner otherwise defaults its tool cache to `<work>/_tool` = `/home/runner/_work/_tool`,
and ARC mounts an `emptyDir` over `/home/runner/_work` — anything baked at the default path
would be hidden at runtime, with no error and no cache hits.

## Upgrading

Bump the relevant `ARG` in [`docker/Dockerfile`](docker/Dockerfile) and open a PR. Each install
step ends in a `--version` invocation, so a wrong URL or changed archive layout fails the build
rather than producing a quietly broken image.

When bumping `RUNNER_VERSION`, keep it aligned with the image tag pinned in the internal
deployment repo.

Releases are cut by semantic-release from Conventional Commits on `main`; the tag push builds
and publishes `ghcr.io/qtsone/runner`.
