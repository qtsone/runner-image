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

Every version is a build `ARG` in [`docker/Dockerfile`](docker/Dockerfile), which is the only
place any of them is written down — Renovate bumps them there, so this table names the arg
instead of repeating a number that would go stale on the next merge.

| Tool | Pinned by | Where | Found by |
|---|---|---|---|
| Node.js | `NODE_VERSION` | `${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64` | `actions/setup-node` |
| Python | `PYTHON_RELEASE_VERSION` | `${RUNNER_TOOL_CACHE}/Python/${PYTHON_VERSION}/x64` | `actions/setup-python` |
| uv | `UV_VERSION` | `${RUNNER_TOOL_CACHE}/uv/${UV_VERSION}/x86_64` | `astral-sh/setup-uv` |
| buf | `BUF_VERSION` | `${RUNNER_TOOL_CACHE}/buf/${BUF_VERSION}/x64` | `bufbuild/buf-setup-action` |
| bun | `BUN_VERSION` | `/home/runner/.bun/bin/bun` | `oven-sh/setup-bun` |
| gh | `GH_VERSION` | `/usr/local/bin/gh` | on `PATH` |
| kubectl | `KUBECTL_VERSION` | `/usr/local/bin/kubectl` | on `PATH` |
| kustomize | `KUSTOMIZE_VERSION` | `/usr/local/bin/kustomize` | on `PATH` |
| jq | `JQ_VERSION` | `/usr/local/bin/jq` | on `PATH` |
| yq | `YQ_VERSION` | `/usr/local/bin/yq` | on `PATH` |
| semantic-release | `SEMANTIC_RELEASE_VERSION` | global in the baked Node | on `PATH` |

Python is the one exception to one-arg-one-version: `actions/python-versions` tags its releases
`<python-version>-<build-id>`, so `PYTHON_RELEASE_VERSION` carries the whole tag and the build
derives the plain `PYTHON_VERSION` from it.

Docker CLI and buildx come from the upstream base image and are not re-installed here.
`gh`, `kubectl`, `kustomize`, `jq`, and `yq` (mikefarah's Go implementation) are plain
system CLIs on `PATH` rather than tool-cache entries, so they need no `setup-*` action
and no version pin in the workflow — they are used as-is. When bumping `KUBECTL_VERSION`, keep it within one minor version of the
target cluster's API server (the supported client/server skew range).

`pnpm` is deliberately **not** baked: `pnpm/action-setup` deletes its install directory and
reinstalls from npm on every run, so a pre-installed copy is never used.

The baked Node's `bin` directory is on `PATH`, so `node`, `npm`, and `semantic-release`
work natively without a `setup-node` step — the same as GitHub-hosted runners.
`semantic-release` is installed globally together with `@semantic-release/changelog` and
`@semantic-release/git`: exactly what `qtsone/actions/release` installs at run time (its
`semantic-version` input default plus its default `extra-plugins`). Keep the three
`SEMANTIC_RELEASE_*` args in lockstep with that action when bumping either side — the action
re-runs `npm install -g` on every release job, and that is only a fast no-op while the
versions match. Renovate groups the three into a single PR and never automerges it, so the
action can be bumped in the same breath.

## Getting a cache hit

A baked tool is only used if the workflow asks for a version that matches it. The rules differ
per action, and two of them need the workflow to pin a version:

- `setup-node` and `setup-python` resolve semver **ranges** against the cache, so a
  major-only `node-version: 24` and a minor-only `python-version: "3.12"` keep hitting the
  baked copies across patch bumps, with nothing to update downstream.
- `buf-setup-action` has its own default version, and hits the baked copy only while that
  default and `BUF_VERSION` agree. Pin `version:` to be sure.
- `setup-uv` resolves `latest` to a concrete version *before* checking the cache, so the
  baked copy stops being used the day a newer uv ships. Pin `version:` to `UV_VERSION`.
- `setup-bun` refuses to reuse an existing install whenever the requested version is
  `latest`, `canary`, or unset. Pin `bun-version:` to `BUN_VERSION` or it always re-downloads.

The exact-pin cases are the ones to re-check after a bump: a workflow still pinning the
previous `UV_VERSION` or `BUN_VERSION` keeps working, it just silently downloads again
instead of using the baked copy.

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

### Renovate

Renovate ([`.github/renovate.json5`](.github/renovate.json5)) raises those bumps on its own.
Each tracked `ARG` carries a `# renovate:` annotation naming its datasource and upstream repo;
adding a tool means adding that annotation next to the new `ARG`, and nothing else.

Because `release.yaml` is path-filtered to `docker/**` and semantic-release cuts no release for
`chore`, anything touching the Dockerfile is committed as `fix(docker):` (or `feat(docker):`
for a major) — a `chore`-prefixed bump would merge, produce no tag, and never rebuild the image.
Updates to this repo's own CI stay `chore(deps):`, where no release is wanted.

What that produces:

| PR | Contents | Automerged |
|---|---|---|
| `fix(docker): update baked toolchain` | every baked tool, batched | yes, on green |
| `fix(docker): update semantic-release` | the three `SEMANTIC_RELEASE_*` args | no — see above |
| `fix(docker): update ghcr.io/actions/actions-runner` | the base image alone | yes, on green |
| `chore(deps): …` | this repo's workflows and actions | yes, on green |

Majors always arrive on their own and are never automerged. Releases must also be at least
three days old before Renovate offers them: an automerged bump reaches every runner on its next
image pull with nobody in between, so a yanked upstream release gets time to be yanked.
