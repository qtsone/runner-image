# syntax=docker/dockerfile:1
# https://docs.docker.com/engine/reference/builder/

# Every ARG below carries a `# renovate:` annotation so Renovate can track it; see
# .github/renovate.json5. RUNNER_VERSION needs none — Renovate's dockerfile manager
# resolves it through the FROM below on its own.
ARG RUNNER_VERSION="2.336.0"
FROM ghcr.io/actions/actions-runner:${RUNNER_VERSION}

# node-version + `node` versioning together mean only LTS lines are ever offered: the
# datasource marks non-LTS releases unstable and the versioning refuses a major that has
# not reached its LTS date.
# renovate: datasource=node-version depName=node versioning=node
ARG NODE_VERSION="24.19.0"

# actions/python-versions tags a release <python-version>-<build-id> and names the tarball
# inside it by the python version alone. One pin, with the python version derived from it
# below — as two ARGs, an update to either alone yields a 404 at build time.
# The versioning drops release-candidate tags (3.15.0-rc.1-309...), which do not match, and
# orders the rest by build id so a rebuild of an unchanged python version is still an update.
# renovate: datasource=github-releases depName=actions/python-versions versioning=regex:^(?<major>\d+)\.(?<minor>\d+)\.(?<patch>\d+)-(?<build>\d+)$
ARG PYTHON_RELEASE_VERSION="3.12.13-27650778726"

# renovate: datasource=github-releases depName=astral-sh/uv
ARG UV_VERSION="0.11.32"
# renovate: datasource=github-releases depName=bufbuild/buf extractVersion=^v(?<version>.+)$
ARG BUF_VERSION="1.50.0"
# renovate: datasource=github-releases depName=oven-sh/bun extractVersion=^bun-v(?<version>.+)$
ARG BUN_VERSION="1.3.14"
# renovate: datasource=github-releases depName=cli/cli extractVersion=^v(?<version>.+)$
ARG GH_VERSION="2.96.0"
# renovate: datasource=github-releases depName=kubernetes/kubernetes extractVersion=^v(?<version>.+)$
ARG KUBECTL_VERSION="1.36.3"
# kubernetes-sigs/kustomize also tags kyaml/ and cmd/config/ releases; the prefix in
# extractVersion is what filters those out.
# renovate: datasource=github-releases depName=kubernetes-sigs/kustomize extractVersion=^kustomize/v(?<version>.+)$
ARG KUSTOMIZE_VERSION="5.8.1"
# renovate: datasource=github-releases depName=jqlang/jq extractVersion=^jq-(?<version>.+)$
ARG JQ_VERSION="1.8.2"
# renovate: datasource=github-releases depName=mikefarah/yq extractVersion=^v(?<version>.+)$
ARG YQ_VERSION="4.53.3"

# Keep these three in lockstep with qtsone/actions/release: the action's semantic-version
# input default, plus the resolved versions of its two default extra-plugins. Renovate
# groups them into one un-automergeable PR so the action can be bumped in the same breath.
# renovate: datasource=npm depName=semantic-release
ARG SEMANTIC_RELEASE_VERSION="25.0.9"
# renovate: datasource=npm depName=@semantic-release/changelog
ARG SEMANTIC_RELEASE_CHANGELOG_VERSION="7.0.0"
# renovate: datasource=npm depName=@semantic-release/git
ARG SEMANTIC_RELEASE_GIT_VERSION="11.0.1"

# The runner picks its tool cache from RUNNER_TOOL_CACHE, then RUNNER_TOOLSDIRECTORY,
# then AGENT_TOOLSDIRECTORY, and only then falls back to <work>/_tool
# (actions/runner, Runner.Common/HostContext.cs). That fallback is /home/runner/_work/_tool,
# which ARC mounts an emptyDir over — every tool baked below would be invisible at runtime
# and every setup-* action would silently re-download. Do not remove this.
ENV RUNNER_TOOL_CACHE=/opt/hostedtoolcache
ENV BUN_INSTALL=/home/runner/.bun

# The baked Node's bin dir goes on PATH so node, npm, and the globally installed
# semantic-release below run natively — no setup-node step — matching GitHub-hosted
# runners, where node is always on PATH. A setup-node that installs a different
# version prepends its own path and still wins.
ENV PATH="${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64/bin:${PATH}"

USER root

# Every install below pipes curl into tar. Under the default shell a failed download is
# masked by a tar that happily succeeds on empty input, producing a silently incomplete image.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates curl unzip xz-utils \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# gh, kubectl, and kustomize are plain system CLIs (like the Docker CLI inherited from the
# base image), not setup-* tool-cache entries, so they go on PATH at /usr/local/bin rather
# than under RUNNER_TOOL_CACHE.
RUN echo "Installing GitHub CLI" \
  && curl -fsSL "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_amd64.tar.gz" \
     | tar -xz -C /tmp \
  && install -m 0755 "/tmp/gh_${GH_VERSION}_linux_amd64/bin/gh" /usr/local/bin/gh \
  && rm -rf "/tmp/gh_${GH_VERSION}_linux_amd64" \
  && gh --version

# Keep kubectl within one minor version of the target cluster's API server (the supported
# skew range) when bumping either side.
RUN echo "Installing kubectl" \
  && curl -fsSL "https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl" -o /tmp/kubectl \
  && install -m 0755 /tmp/kubectl /usr/local/bin/kubectl \
  && rm /tmp/kubectl \
  && kubectl version --client

RUN echo "Installing kustomize" \
  && curl -fsSL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz" \
     | tar -xz -C /tmp \
  && install -m 0755 /tmp/kustomize /usr/local/bin/kustomize \
  && rm /tmp/kustomize \
  && kustomize version

RUN echo "Installing jq" \
  && curl -fsSL "https://github.com/jqlang/jq/releases/download/jq-${JQ_VERSION}/jq-linux-amd64" -o /tmp/jq \
  && install -m 0755 /tmp/jq /usr/local/bin/jq \
  && rm /tmp/jq \
  && jq --version

RUN echo "Installing yq" \
  && curl -fsSL "https://github.com/mikefarah/yq/releases/download/v${YQ_VERSION}/yq_linux_amd64" -o /tmp/yq \
  && install -m 0755 /tmp/yq /usr/local/bin/yq \
  && rm /tmp/yq \
  && yq --version

# Own the cache up front and drop to uid 1001 for every download below. A `chown -R` after
# the fact would copy the entire baked tool cache into a new layer and roughly double the
# image. The runtime needs it writable anyway: a setup-* action asked for a version we did
# not bake calls tc.cacheDir() and writes here as uid 1001.
RUN install -d -o 1001 -g 1001 "${RUNNER_TOOL_CACHE}"

USER runner

# setup-* actions resolve a tool as ${RUNNER_TOOL_CACHE}/<name>/<version>/<arch> and only
# trust it when the sibling marker <arch>.complete exists next to the arch directory —
# note the marker is a file beside <arch>, not inside it (@actions/tool-cache find()).
# The <name> and <arch> strings below are the ones each action actually passes to tc.find():
# they are not interchangeable — Python is capitalised, and uv uses x86_64 where the rest use x64.

RUN echo "Installing Node.js into the tool cache" \
  && NODE_DIR="${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64" \
  && mkdir -p "${NODE_DIR}" \
  && curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz" \
     | tar -xJ -C "${NODE_DIR}" --strip-components=1 \
  && touch "${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64.complete" \
  && "${NODE_DIR}/bin/node" --version

# All three packages must go in one npm invocation: the plugins then land as siblings of
# semantic-release in the global node_modules, which is where its plugin loader resolves
# them from when the repo has no local install (it walks up from semantic-release's own
# directory). Installed separately they would not be resolvable.
RUN echo "Installing semantic-release into the baked Node" \
  && npm install -g \
       "semantic-release@${SEMANTIC_RELEASE_VERSION}" \
       "@semantic-release/changelog@${SEMANTIC_RELEASE_CHANGELOG_VERSION}" \
       "@semantic-release/git@${SEMANTIC_RELEASE_GIT_VERSION}" \
  && rm -rf /home/runner/.npm \
  && semantic-release --version

# The python-versions tarball ships the same setup.sh GitHub's own images run: it lays out
# the arch directory, creates the python/pythonX.Y symlinks setup-python expects, upgrades
# pip, and writes the .complete marker. Reuse it rather than reproducing that layout by hand.
RUN echo "Installing Python into the tool cache" \
  && PYTHON_VERSION="${PYTHON_RELEASE_VERSION%%-*}" \
  && mkdir -p /tmp/python \
  && curl -fsSL "https://github.com/actions/python-versions/releases/download/${PYTHON_RELEASE_VERSION}/python-${PYTHON_VERSION}-linux-24.04-x64.tar.gz" \
     | tar -xz -C /tmp/python \
  && cd /tmp/python \
  && bash ./setup.sh \
  && cd / \
  && rm -rf /tmp/python \
  && "${RUNNER_TOOL_CACHE}/Python/${PYTHON_VERSION}/x64/bin/python3" --version

RUN echo "Installing uv into the tool cache" \
  && UV_DIR="${RUNNER_TOOL_CACHE}/uv/${UV_VERSION}/x86_64" \
  && mkdir -p "${UV_DIR}" \
  && curl -fsSL "https://github.com/astral-sh/uv/releases/download/${UV_VERSION}/uv-x86_64-unknown-linux-gnu.tar.gz" \
     | tar -xz -C "${UV_DIR}" --strip-components=1 \
  && touch "${RUNNER_TOOL_CACHE}/uv/${UV_VERSION}/x86_64.complete" \
  && "${UV_DIR}/uv" --version

# buf-setup-action adds <cachedir>/bin to PATH, so the binary has to sit under bin/.
RUN echo "Installing buf into the tool cache" \
  && BUF_DIR="${RUNNER_TOOL_CACHE}/buf/${BUF_VERSION}/x64" \
  && mkdir -p "${BUF_DIR}" \
  && curl -fsSL "https://github.com/bufbuild/buf/releases/download/v${BUF_VERSION}/buf-Linux-x86_64.tar.gz" \
     | tar -xz -C "${BUF_DIR}" --strip-components=1 \
  && touch "${RUNNER_TOOL_CACHE}/buf/${BUF_VERSION}/x64.complete" \
  && "${BUF_DIR}/bin/buf" --version

# setup-bun ignores the tool cache and any bun on PATH: it looks only at ~/.bun/bin/bun
# and compares that binary's own revision against the requested version. Installing
# elsewhere would be dead weight.
RUN echo "Installing bun" \
  && mkdir -p "${BUN_INSTALL}/bin" \
  && curl -fsSL "https://github.com/oven-sh/bun/releases/download/bun-v${BUN_VERSION}/bun-linux-x64.zip" -o /tmp/bun.zip \
  && unzip -q /tmp/bun.zip -d /tmp/bun \
  && mv /tmp/bun/bun-linux-x64/bun "${BUN_INSTALL}/bin/bun" \
  && chmod +x "${BUN_INSTALL}/bin/bun" \
  && ln -s bun "${BUN_INSTALL}/bin/bunx" \
  && rm -rf /tmp/bun /tmp/bun.zip \
  && "${BUN_INSTALL}/bin/bun" --version

# The upstream image sets no ENTRYPOINT and defaults CMD to /bin/bash, so a pod that loses
# its `command:` override silently runs a shell, reads EOF on a TTY-less stdin, and exits 0 —
# the runner never registers and the pod churns forever with no logs. Setting ENTRYPOINT here
# makes that failure mode impossible regardless of what the Helm values do.
ENTRYPOINT ["/home/runner/run.sh"]
