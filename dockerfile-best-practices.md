# Dockerfile Best Practices: From Base Image and Cache to SBOM, Cosign, and CI/CD

*Translated from the original Russian article by Seyd-Magomed Kuzgov (@casssuzy), published on Habr: [https://habr.com/ru/articles/1041784/](https://habr.com/ru/articles/1041784/)*

**Difficulty:** Intermediate · **Reading time:** ~30 min · **Tags:** dockerfile, docker, best practice, kubernetes, images, containers, devops, devsecops, containerization

---

## Foreword

This article turned out to be long: there are many practices, and each one matters in its own way. I put it together as a set of best practices — not every point is needed for every project, but almost every point comes up sooner or later in a review, in CI, or after an unpleasant incident.

I tried to write for different experience levels: from basic mistakes like `COPY . .`, `latest`, and running as root, to production-grade topics like BuildKit, secrets, SBOM, image signing, and software supply chain protection.
So the tone here is intentionally dry, direct, and engineering-focused: no long preambles, no filler, and no retelling of documentation just for the sake of it. My goal wasn't to write an overview article, but a working reference you can come back to when writing, reviewing, or improving a Dockerfile.

To make the article easier to navigate, I split it into thematic blocks. Below is the table of contents — click a section to jump straight to it.

**Table of contents:**

1. [Base image, versions, and controlled updates](#1-base-image-versions-and-controlled-updates)
2. [Build context, `.dockerignore`, copying files, and safely fetching external data](#2-build-context-dockerignore-copying-files-and-safely-fetching-external-data)
3. [Layers, instruction order, caching, and BuildKit](#3-layers-instruction-order-caching-and-buildkit)
4. [Multi-stage builds and language-specific optimizations](#4-multi-stage-builds-and-language-specific-optimizations)
5. [Secrets: build-time, runtime, and protecting against leaks in layers](#5-secrets-build-time-runtime-and-protecting-against-leaks-in-layers)
6. [User, UID, file permissions, and container immutability](#6-user-uid-file-permissions-and-container-immutability)
7. [Process startup: CMD, ENTRYPOINT, PID 1, SIGTERM, and the "one container, one service" model](#7-process-startup-cmd-entrypoint-pid-1-sigterm-and-the-one-container-one-service-model)
8. [Runtime behavior: healthchecks, ports, volumes, configs, logs, resources, and Gunicorn specifics](#8-runtime-behavior-healthchecks-ports-volumes-configs-logs-resources-and-gunicorn-specifics)
9. [Software supply chain, registries, signing, scanning, linting, testing, and CI/CD](#9-software-supply-chain-registries-signing-scanning-linting-testing-and-cicd)
10. [Final Dockerfile examples](#10-final-dockerfile-examples)
11. [Conclusion](#conclusion)

***I'd also genuinely welcome your additions in the comments: practical cases, debatable points, personal experience, mistakes you've run into in real-world operation, and alternative approaches. I read feedback and update the material when needed — clarifying wording, fixing inaccuracies, and adding useful notes if they make the article stronger and more accurate.***

### Why think about the Dockerfile at all

A Dockerfile isn't just an instruction for how to run an application. It's a description of your future production environment: which packages end up inside, which user the process runs as, which secrets might accidentally end up in the layers, how quickly the image builds in CI/CD, and how predictably it will behave a month from now.

A good Dockerfile needs to solve three problems at once:

1. **Reproducibility** — the same version of a Dockerfile should produce a predictable image.
2. **Minimal size and fast builds** — the image should contain only what the application actually needs.
3. **Security** — less unnecessary software, fewer privileges, fewer chances of secret leaks, and easier vulnerability control.

---

## 1. Base image, versions, and controlled updates

The base image is set with the `FROM` instruction, and it has the biggest impact on the size, security, build speed, and overall convenience of the resulting container. A bad habit is grabbing a full Ubuntu/Debian/CentOS "just in case," and then ending up with a pile of unnecessary utilities, libraries, and potential vulnerabilities inside the container.

The rule is simple: **use the smallest and most appropriate base image that can actually run your application**.

Bad:

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 python3-pip curl vim git
```

Better:

```dockerfile
FROM python:3.12-slim
```

Even better for some applications:

```dockerfile
FROM gcr.io/distroless/python3-debian12
```

### 1.1. Choose an image specialized for the job

For a Node.js application, use `node`; for Python, use `python`; for Java, use `eclipse-temurin`, `amazoncorretto`, or another supported JDK/JRE image; for Nginx, use `nginx`; for PostgreSQL, use `postgres`.

Specialized images already contain the runtime you need, so you don't have to manually assemble an environment from random packages. This reduces size, cuts down on unnecessary dependencies, and makes the image easier to maintain.

When choosing, look at four things:

- The image should be official or come from a trusted vendor.
- It should be updated regularly.
- It should have a clear Dockerfile or a clear supply chain.
- It should be small enough, but not at the cost of stability.

### 1.2. Use slim, alpine, distroless, and scratch deliberately

The smallest image isn't always the most minimal one in practice. Every option has a trade-off.

#### slim

`slim` images are usually based on Debian but contain fewer unnecessary packages. An important advantage is that they still ship with `glibc`, which often makes them a better fit for Python, Java, Node.js, and applications with native dependencies.

This is often the best compromise:

```dockerfile
FROM python:3.12-slim
```

#### alpine

Alpine is very small but uses `musl` instead of `glibc`. Because of this, some Python/C/C++ dependencies may build more slowly, behave differently, or require extra packages.

Alpine works well when:

- the application has already been verified on Alpine;
- the dependencies don't conflict with `musl`;
- image size is genuinely critical.

But Alpine shouldn't be the automatic default just because it's small.

#### distroless

Distroless images contain a runtime and a minimal set of libraries, but almost no system tools: no shell, package manager, curl, vim, etc. This reduces the attack surface — if an attacker gets inside the container, they have fewer ready-made tools to work with.

The downside is that debugging is harder: you can't just exec into the container and run `bash`.

Example for Go:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app ./cmd/app

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

#### scratch

`scratch` is a completely empty image. It only works for statically built binaries that don't need a shell, libc, certificates, timezone data, or other system files.

```dockerfile
FROM scratch
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

Only use `scratch` if you understand exactly which files your application needs at runtime.

### 1.3. Don't use `latest` in production

`latest` has nothing to do with a stable version. It's just a tag that can change depending on updates. Today `node:latest` might point to one version of Node.js; tomorrow, to a different one. In CI/CD this turns into a lottery: the Dockerfile hasn't changed, but the build suddenly breaks.

Bad:

```dockerfile
FROM node:latest
```

Better:

```dockerfile
FROM node:24.16.0-slim
```

Even stricter:

```dockerfile
FROM node@sha256:<digest>
```

For production it's better to pin:

- the runtime version: `node:24.16.0`, `python:3.12.13`, `golang:1.22.2`;
- the image variant: `slim`, `alpine`, `bookworm`, `bullseye`;
- and, for stricter requirements, the digest via `sha256`.

A tag can be overwritten; a digest can't. That's why a digest gives you maximum reproducibility.

### 1.4. Update base images deliberately, not randomly

Avoiding `latest` doesn't mean "never update." On the contrary: images need to be regularly rebuilt and updated to get security patches. The difference is that the update should be controlled.

Good strategy:

- use stable/LTS versions;
- track the end-of-support date for the chosen version;
- rebuild images regularly;
- scan them for CVEs;
- update the digest or version after verification.

Bad strategy:

```dockerfile
FROM python:latest
```

Good strategy:

```dockerfile
FROM python:3.12.13-slim-bookworm
```

And then, as a separate process, update to the next current patch release or a new digest, run tests and scans, and only then roll it out.

### 1.5. Don't auto-upgrade all system packages

Commands like `apt-get upgrade`, `yum update`, or `apk upgrade` inside a Dockerfile often make the build less predictable. Today they install one set of packages, tomorrow another. On top of that, you might silently end up with new components that haven't been checked by SCA tools or scanners.

This isn't to say security patches aren't needed — they are. But it's better to get them by updating the base image, rebuilding regularly, and updating in a controlled way, rather than through a random `apt-get upgrade` in every build with no pinning and no verification.

Bad:

```dockerfile
RUN apt-get update && apt-get upgrade -y
```

Better:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

In environments with strict requirements, pin package versions:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    cowsay=3.03+dfsg1-6 \
    && rm -rf /var/lib/apt/lists/*
```

If a package pulls in a dependency, that dependency needs to be accounted for in vulnerability analysis too.

### 1.6. Install only the packages you actually need

Every installed package adds:

- extra size;
- new CVEs;
- more build and pull/push time;
- more tools inside the container that an attacker could use.

So don't put `vim`, `git`, `gcc`, `make`, `curl`, `wget`, or `bash` into a runtime image if the application works without them.

For Debian/Ubuntu, almost always use:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

`--no-install-recommends` stops the package manager from installing recommended-but-not-required dependencies.

---

## 2. Build context, `.dockerignore`, copying files, and safely fetching external data

This block covers everything that ends up in the image from outside: project files, archives, dependencies, external URLs, secrets in the build context, and the choice between `COPY` and `ADD`. The core idea: **only what's needed should end up in the image, from a clear source, fetched in a verifiable way**.

### 2.1. Use `.dockerignore`

Before building, Docker sends the build context to the daemon. If the context is the project root, that can include `.git`, `node_modules`, `.env`, keys, logs, caches, build artifacts, and local IDE settings.

`.dockerignore` solves several problems at once:

- shrinks the build context;
- speeds up the build;
- reduces the risk of secret leaks;
- prevents unnecessary cache invalidation;
- stops junk from accidentally being copied into the image.

Example of a basic `.dockerignore`:

```
.git
.gitignore
.vscode/
.idea/
.env
.env.*
*.log
__pycache__/
*.pyc
node_modules/
coverage/
dist/
build/
.cache/
.DS_Store
.aws/
.ssh/
```

For Go/Java/compiled projects, an allowlist approach is often better: ignore everything first, then explicitly allow what's needed.

```
*
!go.mod
!go.sum
!cmd/
!internal/
!pkg/
```

But it's important not to overdo it: `.dockerignore` should match the language and build process. If a Java image only needs the built `.jar`, it's better to copy just that rather than the whole project.

### 2.2. Don't copy the whole project unless you need to

Anti-pattern:

```dockerfile
COPY . .
```

Sometimes this is fine, but often it's too broad. This command copies everything left in the build context after `.dockerignore` filtering. If `.dockerignore` is incomplete, extra files will end up in the image.

It's better to copy explicitly:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
```

Or for Java:

```dockerfile
COPY target/app.jar /app/app.jar
```

The point: a Dockerfile should copy exactly what's needed for the build or for running the app.

### 2.3. Use `COPY` by default, and `ADD` only when you actually need its extra capabilities

`COPY` is the best default choice when you just need to move local files from the build context into the image. It does one clear thing: copies files and directories. So for source code, lock files, configs, built `.jar` files, binaries, and other local artifacts, `COPY` is almost always what you want.

```dockerfile
COPY package.json package-lock.json ./
COPY src/ ./src/
```

A Dockerfile like this is easier to read and review: it's immediately clear that we're taking files from the build context and placing them into the image. There's no network download, automatic extraction, or implicit logic involved.

`ADD` can do more:

- copy local files;
- extract local tar archives;
- download files from a URL;
- work with Git sources;
- verify checksums for remote resources;
- control tar extraction via `--unpack`.

Because of this extra functionality, `ADD` shouldn't be used as a substitute for `COPY` for ordinary copying.

A bad example — using `ADD` for a remote file without verification:

```dockerfile
ADD https://example.com/app.tar.gz /app
```

A Dockerfile like this doesn't tell the reader which exact version of the artifact we expect to get, and it doesn't verify its checksum. The source is technically specified, but trust in the content remains implicit.

It's better if the remote artifact is public and you have a fixed SHA-256 checksum:

```dockerfile
# syntax=docker/dockerfile:1

ARG TOOL_VERSION=1.2.3

ADD --checksum=sha256:<expected_sha256> \
    --unpack=true \
    https://example.com/tool-${TOOL_VERSION}.tar.gz /usr/local/bin/
```

In this case, `ADD` does exactly what's needed: it downloads the remote artifact, verifies its checksum, and, if it's a tar archive, extracts it to the right location. There's no need to install `curl`, `tar`, `sha256sum`, and a shell in the build stage just for a single download.

This approach has another benefit: BuildKit better understands the remote artifact as a separate build input. This helps with caching and is especially noticeable in multi-platform builds, where extra commands inside `RUN` can suddenly start running through QEMU for a non-native architecture.

But `ADD --checksum` / `ADD --unpack=true` isn't a universal replacement for `RUN`.

If the download requires authorization, tokens, custom headers, GPG verification, or more complex logic, `RUN` is still the right choice. In that case, though, secrets shouldn't be passed via `ARG` or `ENV` — it's better to use BuildKit secrets.

So the rule comes out as:

- local files from the build context — `COPY`;
- a local tar archive you deliberately want to extract — `ADD`;
- a public remote archive with a fixed SHA-256 checksum — `ADD --checksum` and, if needed, `ADD --unpack=true`;
- a private download, tokens, custom headers, GPG verification, or complex logic — `RUN` combined with BuildKit secrets.

The main point isn't that `ADD` is bad. What's bad is an implicit, unverified data source. If the source is clear, the checksum is fixed, and the Dockerfile's behavior is obvious, `ADD` can absolutely be the right tool.

### 2.4. Download external files safely

A dangerous anti-pattern:

```dockerfile
RUN curl -fsSL http://example.com/install.sh | sh
```

There are three problems here at once:

1. an insecure channel or unverified source is used;
2. the downloaded script is executed immediately;
3. there's no signature or checksum verification.

If it's a public archive and SHA-256 verification is sufficient, use `ADD --checksum` and, if needed, `ADD --unpack=true`, as in the example above. This way the Dockerfile clearly shows the external source, the expected checksum, and the extraction behavior.

If the download requires a token, non-standard headers, GPG verification, or any additional logic, `ADD` won't always fit. In that case it's better to keep an explicit `RUN`: download the artifact, verify it, and remove temporary files within the same instruction. Secrets shouldn't be passed via `ARG` or `ENV` here either — it's safer to mount them only for the duration of the build, via BuildKit.

```dockerfile
# syntax=docker/dockerfile:1

ARG TOOL_VERSION=1.2.3
ARG TOOL_SHA256=<expected_sha256>

RUN --mount=type=secret,id=download_token <<'EOF'
set -eu

TOKEN="$(cat /run/secrets/download_token)"

curl -fsSLo /tmp/tool.tar.gz \
  -H "Authorization: Bearer ${TOKEN}" \
  "https://example.com/tool-${TOOL_VERSION}.tar.gz"

echo "${TOOL_SHA256}  /tmp/tool.tar.gz" | sha256sum -c -
tar -xzf /tmp/tool.tar.gz -C /usr/local/bin
rm -f /tmp/tool.tar.gz
EOF
```

Two things matter here: the secret only exists for the duration of that specific `RUN`, and the temporary archive is removed within the same instruction that created it.

If you're adding a third-party apt/yum/apk repository, verify its GPG keys and source. Don't turn your Dockerfile into a chain of trust pointing at a random URL.

---

## 3. Layers, instruction order, caching, and BuildKit

A Docker image consists of layers. The `RUN`, `COPY`, and `ADD` instructions each create a new layer. If a layer changes, it and everything below it gets rebuilt. So instruction order, combining commands, and how you handle caching directly affect build speed, image size, and security.

### 3.1. Order instructions so the cache actually works

A common mistake is copying all the code first and only then installing dependencies.

Bad:

```dockerfile
COPY . .
RUN npm install
```

Any change in the code invalidates the dependency-installation cache.

Better:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

General principle:

1. Base image first.
2. Then system dependencies.
3. Then lock files and dependency manifests.
4. Then dependency installation.
5. Then source code.
6. At the very end — whatever changes most often.

For Python:

```dockerfile
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

For Go:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app ./cmd/app
```

For Node.js:

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

### 3.2. Combine related commands into a single RUN

If you create extra junk in one layer and remove it in another, the image size can still stay large — the data has already landed in a lower layer.

Bad:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Better:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

Rule: **if you created a temporary file, a cache, or a package index — remove it in the same `RUN` instruction**.

For different package managers:

```dockerfile
# Debian/Ubuntu
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# Alpine
RUN apk add --no-cache curl

# Yum/DNF
RUN dnf install -y curl \
    && dnf clean all \
    && rm -rf /var/cache/dnf
```

### 3.3. Minimize layers, but don't turn the Dockerfile into an unreadable monolith

Reducing the number of layers is useful, but it's not the only goal. If you cram the whole Dockerfile into one giant `RUN`, it becomes hard to read, maintain, and cache.

A good approach:

- combine logically related commands;
- keep steps separate when it's worth caching them individually;
- don't spend too much time micro-optimizing a builder stage if it doesn't end up in the final image;
- sort package lists alphabetically for readability and cleaner diffs.

Example:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*
```

### 3.4. Check what's actually sitting in the layers

Don't trust the feeling that you deleted a file. Check the image.

Useful commands:

```bash
docker history my-image:tag
```

```bash
docker image inspect my-image:tag
```

It's also handy to use layer-analysis tools like `dive`. They show which files were added, changed, or removed in each layer, and help you find junk, secrets, caches, and heavy dependencies.

### 3.5. Use BuildKit

BuildKit is Docker's modern build backend. It can build the dependency graph more efficiently, parallelize independent stages, and use cache mounts, secret mounts, and bind mounts.

You can enable it like this:

```bash
DOCKER_BUILDKIT=1 docker build -t myapp .
```

Or use `docker buildx`.

#### Cache mounts

BuildKit lets you cache dependencies without baking the cache into the image layer.

Python:

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

Node.js:

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

Apt:

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends curl
```

#### Bind mounts at build time

Sometimes a file is only needed for the build command but shouldn't end up in the image. BuildKit lets you mount it temporarily.

For example, instead of copying `requirements.txt` into the runtime stage, you can use a bind mount:

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt,readonly \
    --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r /tmp/requirements.txt
```

This helps reduce the number of layers and avoids leaving extra files in the image.

#### Heredoc for complex RUN instructions

When a `RUN` instruction has one or two commands, the usual style with `&&` and line continuations reads fine:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

But once the build command gets longer, with cache mounts, secret mounts, bind mounts, several checks, and conditional logic, the Dockerfile quickly turns into a pile of backslashes. In these places, a heredoc is more convenient:

```dockerfile
# syntax=docker/dockerfile:1

RUN --mount=type=cache,target=/root/.cache/pip <<EOF
set -eux

python -m pip install --upgrade pip
pip install -r requirements.txt
python -m compileall /app
EOF
```

A heredoc doesn't make the build secure by itself, but it makes complex build steps more readable. That matters not just for aesthetics: this kind of code is easier to review, easier to change, and easier to debug.

Balance matters here too. If a `RUN` already contains a full-blown script with dozens of lines, functions, and complex conditionals, it's probably better to move it into a separate script and explicitly copy it into the builder stage. But for medium-sized build steps, heredoc is often much more convenient than the classic `\`-continuation style.

### 3.6. Use caching correctly in CI/CD

Without a cache, Docker builds in CI can be slow. But the cache shouldn't compromise security or reproducibility.

Practices:

- use the BuildKit/buildx cache;
- use `--cache-from` and `--cache-to` if the runner is ephemeral;
- cache dependencies via `--mount=type=cache`;
- don't cache secrets;
- don't rely on the cache as the only way to get current security patches;
- periodically do a clean rebuild and scan.

Example with buildx:

```bash
docker buildx build \
  --cache-from=type=registry,ref=registry.example.com/myapp:buildcache \
  --cache-to=type=registry,ref=registry.example.com/myapp:buildcache,mode=max \
  -t registry.example.com/myapp:${GIT_SHA} \
  --push \
  .
```

### 3.7. Consider Podman/Buildah as an alternative to Docker in CI

In CI, classic Docker-in-Docker isn't always convenient or safe to use. In these cases, you might consider `podman`/`buildah`: they can build images from Dockerfile/Containerfile syntax and often fit better into rootless scenarios.

Working with registry-based cache via `--cache-from` and `--cache-to` is also worth highlighting on its own. The idea: layer cache is stored not just locally on the runner, but in a registry. This is especially useful when CI runners are ephemeral and every pipeline starts from a clean machine.

Example for Podman/Buildah:

```bash
podman build \
  --layers \
  --cache-from registry.example.com/myapp/buildcache \
  --cache-to registry.example.com/myapp/buildcache \
  -t registry.example.com/myapp:${GIT_SHA} \
  .
```

A similar approach exists in Docker Buildx/BuildKit via `--cache-from` and `--cache-to`. For CI, what matters isn't so much "Docker vs. Podman," but support for portable caching that can be stored in a registry and reused across pipelines.

When Podman/Buildah may be especially appropriate:

- in rootless CI builds;
- when you don't want to run a Docker daemon inside the job;
- when the infrastructure already uses Podman;
- when a daemonless approach is needed;
- when the cache needs to live in a registry and be reused across ephemeral runners.

### 3.8. When `docker buildx build` isn't enough anymore: `buildx bake`

For a simple service, a plain `docker buildx build` command is usually enough. But over time it can turn into a long line with platforms, targets, caches, arguments, tags, SBOM, provenance, and various image variants.

In these cases, it's worth looking at `docker buildx bake`. It's a way to describe a build declaratively: instead of hand-building a huge CLI command, you move targets, platforms, arguments, and caches into a separate file.

Minimal example:

```hcl
variable "GIT_SHA" {
  default = "dev"
}

group "default" {
  targets = ["app"]
}

target "app" {
  dockerfile = "Dockerfile"
  context    = "."
  platforms  = ["linux/amd64", "linux/arm64"]
  tags       = ["registry.example.com/myapp:${GIT_SHA}"]

  cache-from = ["type=registry,ref=registry.example.com/myapp:buildcache"]
  cache-to   = ["type=registry,ref=registry.example.com/myapp:buildcache,mode=max"]
}
```

Run it with:

```bash
GIT_SHA=${GIT_SHA} docker buildx bake
```

`bake` is especially useful when you need to build several image variants: a regular runtime image, a debug image, a sanitizer-enabled image, separate targets for tests, multiple platforms, or several related artifacts.

But for a small project this can be unnecessary complexity. If the whole build fits into one clear `docker buildx build` command, there's no need to drag in `bake` just for the sake of it. It starts paying off once the build has become its own configuration rather than a single command in CI.

---

## 4. Multi-stage builds and language-specific optimizations

A multi-stage build separates the build environment from the runtime environment. The builder stage can contain compilers, SDKs, dev dependencies, caches, and source code. The final image should only contain the runtime and the artifacts that are actually needed.

This practice simultaneously reduces image size, lowers the attack surface, and keeps the runtime image cleaner.

### 4.1. Use multi-stage builds

Bad:

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o app ./cmd/app
CMD ["./app"]
```

This leaves the Go toolchain, source code, and other unnecessary files in the image.

Better:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app ./cmd/app

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Multi-stage builds are especially useful for:

- Go, Rust, C/C++;
- Java applications, where the build stage produces a jar;
- Python, when you need to build wheels or a venv;
- Node.js, when you need to separate dev dependencies from production dependencies;
- frontend builds, where Node.js is only needed to build static files.

### 4.2. For Python, build wheels or carry over a virtualenv from the builder stage

In Python projects, some dependencies require compilation. The compiler is needed during the build, but not at runtime.

Wheel-file approach:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY . .
CMD ["python", "main.py"]
```

Virtualenv approach:

```dockerfile
FROM python:3.12-slim AS builder
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
ENV PATH="/opt/venv/bin:$PATH"
COPY --from=builder /opt/venv /opt/venv
WORKDIR /app
COPY . .
CMD ["python", "main.py"]
```

A virtualenv inside a container usually isn't necessary, because the container already isolates the environment. But in a multi-stage build it can be convenient: you can build the environment in the builder stage and carry it over wholesale into the runtime stage.

### 4.3. Speed up Python builds with modern tools, but don't sacrifice predictability

For Python you can use `uv` and other fast dependency installers. This speeds up builds, especially in CI. But the speedup shouldn't override the basic rules:

- pin dependencies with a lock file;
- use the BuildKit cache;
- don't bake the cache into the final image;
- separate build dependencies from runtime dependencies;
- don't copy secrets or unnecessary files.

Example idea:

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install --system -r requirements.txt
```

### 4.4. For multi-platform builds, don't run the build stage through QEMU unnecessarily

Multi-stage builds separate building from running, but multi-platform builds introduce another trap. If you build an image for several architectures at once — say, `linux/amd64` and `linux/arm64` — the builder may start running non-native stages through emulation. It will work, but sometimes very slowly.

This is especially painful in compiled projects: Go, Rust, C, C++, and sometimes Java with native dependencies. The architecture native to the runner builds quickly, while the other one suddenly ends up running inside QEMU and turns the pipeline into a long wait.

If the language and build system support cross-compilation, it's better to run the build stage on the builder's native architecture, and compile the artifact for the target architecture from within it.

Example for Go:

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.22 AS builder

WORKDIR /src

ARG TARGETOS
ARG TARGETARCH

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/root/.cache/go-build <<EOF
set -eux

CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
  go build -o /out/app ./cmd/app
EOF

FROM gcr.io/distroless/static-debian12

COPY --from=builder /out/app /app

USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

The key line here is:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS builder
```

This forces the builder stage to run on the platform of the machine doing the build. `TARGETOS` and `TARGETARCH` are then used to compile for the actual target platform.

The idea is simple: the final image needs to target the right platform, but the heavy build logic should run natively whenever possible. For small projects the difference may be negligible, but for large C/C++/Rust/Go builds it can sometimes be the difference between a few minutes and an hour of waiting.

---

## 5. Secrets: build-time, runtime, and protecting against leaks in layers

Secrets must never be passed via `ENV`, `ARG`, `COPY`, `RUN echo ...`, and must never be left in files that end up in the build context.

Bad:

```dockerfile
ARG SSH_PRIVATE_KEY
ENV API_TOKEN=super-secret
COPY . .
```

Why this is bad:

- the secret can end up in a layer;
- the secret can be visible via `docker history`;
- environment variables can be seen via inspect/at runtime;
- deleting a file in a later layer doesn't remove it from the earlier layer.

### 5.1. BuildKit secrets for the build stage

```dockerfile
# syntax=docker/dockerfile:1.8
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN="$(cat /run/secrets/npm_token)" npm ci
```

Build command:

```bash
docker build \
  --secret id=npm_token,src=.npm_token \
  -t myapp .
```

The secret is mounted only for the duration of that specific `RUN` and isn't persisted into a layer.

### 5.2. Runtime secrets

For runtime, use:

- Docker secrets;
- Kubernetes Secrets;
- a Secret Manager/Vault;
- environment variables, only if that's acceptable for your threat model;
- mounted secret files.

And always exclude `.env`, `.aws`, `.ssh`, private keys, and local configs via `.dockerignore`.

---

## 6. User, UID, file permissions, and container immutability

This block brings together practices related to the principle of least privilege. A container shouldn't run as root unless it has to, an application shouldn't be able to rewrite its own code, and the filesystem should be structured so the container can run across different runtime environments.

### 6.1. Run the application as a non-root user

By default, a container often runs its process as root. That's convenient but bad for security. If the application is compromised, root inside the container increases the risk of escalation — especially with volumes, the Docker socket, capabilities, or configuration mistakes.

Create a user and switch to it:

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

For Debian/Ubuntu:

```dockerfile
RUN groupadd -r app && useradd -r -g app -d /nonexistent -s /usr/sbin/nologin app
USER app
```

For Alpine:

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

If the base image already includes a non-root user, use it. For example, Node.js images often come with a `node` user.

### 6.2. Don't hard-code yourself to a single UID

In some environments, such as OpenShift, a container can run with an arbitrary UID. If a Dockerfile is built assuming a specific user and a specific UID, the application may not have access to the directories it needs.

Bad:

```dockerfile
RUN mkdir /app-tmp && chown -R app:app /app-tmp
USER app
ENV TMP_DIR=/app-tmp
```

If the runtime starts the container with a different UID, writing to `/app-tmp` can break.

Better:

```dockerfile
ENV TMP_DIR=/tmp
```

Practices:

- write temporary files to `/tmp` where appropriate;
- make application files readable if that's safe;
- don't require file ownership just to run the app;
- test running with an arbitrary UID;
- don't solve permission problems by running as root.

### 6.3. Make executable files immutable for the runtime user

You don't always need to make the application's runtime user the owner of the code. The app often only needs read and execute permissions.

Bad pattern:

```dockerfile
COPY --chown=app:app . /app
USER app
```

If the application — or an attacker — gets write access to `/app`, they can modify executables or entrypoint scripts.

Better:

```dockerfile
COPY . /app
RUN chmod -R a-w /app && chmod +x /app/entrypoint.sh
USER app
```

The idea: the code and executables are owned by root; the runtime user can read/execute them but not modify them. Directories that need to be writable are created explicitly: `/tmp`, `/var/cache/myapp`, `/data`, etc.

### 6.4. If root actions are needed at startup, use gosu/su-exec, not sudo

Sometimes the entrypoint needs to perform a root action first — for example, fixing volume ownership — and then start the application as a regular user.

In that case, don't launch the application itself through `sudo`. It's better to use `gosu` or `su-exec`, because they don't create an extra process chain and help preserve the "one main process" model.

Example:

```bash
#!/bin/sh
set -e

if [ "$1" = "myapp" ]; then
  chown -R app:app /data
  exec gosu app "$@"
fi

exec "$@"
```

Important: `gosu` isn't a universal replacement for `sudo`. It's appropriate specifically in entrypoint scenarios, where you need to perform minimal root actions and then replace the process with the application.

---

## 7. Process startup: CMD, ENTRYPOINT, PID 1, SIGTERM, and the "one container, one service" model

A container should start up correctly, accept stop signals, terminate without losing state, and not turn into a mini virtual machine running several independent services inside.

### 7.1. Use the exec form of CMD and ENTRYPOINT

Docker supports two forms.

Shell form:

```dockerfile
CMD "python app.py"
```

Exec form:

```dockerfile
CMD ["python", "app.py"]
```

The exec form is almost always the better choice. With the shell form, Docker runs `/bin/sh -c ...`, and the shell becomes PID 1. Because of this, signals may not reach the application, and container shutdown becomes less predictable.

Bad:

```dockerfile
ENTRYPOINT python app.py
```

Better:

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

### 7.2. Understand the difference between CMD and ENTRYPOINT

`ENTRYPOINT` is the container's main command. It's harder to accidentally override.

`CMD` is the default command, or default arguments. It's easy to replace at `docker run` time.

A good pattern:

```dockerfile
ENTRYPOINT ["gunicorn", "config.wsgi", "-b", "0.0.0.0:8000", "-w"]
CMD ["4"]
```

By default this runs:

```
gunicorn config.wsgi -b 0.0.0.0:8000 -w 4
```

And at runtime you can change just the argument:

```bash
docker run myapp 8
```

In other words, `ENTRYPOINT` defines what to run, and `CMD` defines the default parameters.

### 7.3. Handle SIGTERM and PID 1 correctly

In Kubernetes and Docker, when a container is stopped it first receives SIGTERM, and then, once the grace period for clean shutdown ends, SIGKILL. If the application doesn't receive SIGTERM, it can't properly close connections, finish in-flight requests, persist state, or release resources.

A common problem:

```dockerfile
ENTRYPOINT ["/app/start.sh"]
```

And inside `start.sh`:

```bash
python app.py
```

In this case the shell script remains PID 1 and may not forward the signal to the application.

Correct version:

```bash
#!/bin/sh
set -e
exec python app.py
```

`exec` replaces the shell with the application process, and the application becomes PID 1.

If the application needs an init process, use `tini` or a similar minimal init that forwards signals and reaps zombie processes.

### 7.4. Run a single main process per container

A container is best designed around a single service: one web server, one worker, one database process, and so on. This makes the following easier:

- scaling;
- logging;
- healthchecks;
- graceful shutdown;
- updates;
- reuse;
- debugging.

Bad: one container running Nginx, the application, cron, and a database all at once.

Better: a separate container for each service, connected via the network, volumes, a queue, or an orchestrator.

There are exceptions: sidecar patterns, init processes, and tightly coupled helper processes. But that should be a deliberate architectural decision, not a habit of stuffing everything into one container.

---

## 8. Runtime behavior: healthchecks, ports, volumes, configs, logs, resources, and Gunicorn specifics

The Dockerfile is only part of the story. The image needs to behave well at runtime too: respond to checks, not expose unnecessary entry points, not store mutable data inside itself, write logs outward, and run correctly under CPU/memory limits.

### 8.1. Add a HEALTHCHECK if the image runs under Docker/Docker Swarm

Docker considers a container alive as long as its main process is alive. But the process can hang, deadlock, stop responding, or return 500s.

Example:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8000/health || exit 1
```

Or:

```dockerfile
HEALTHCHECK CMD curl --fail http://127.0.0.1:8000/health || exit 1
```

But don't install `curl` just for the healthcheck if you can check things using the application itself or a minimal utility already present in the image.

For Kubernetes, keep in mind: a Dockerfile `HEALTHCHECK` doesn't replace `livenessProbe`, `readinessProbe`, or `startupProbe`. In Kubernetes, checks are better described in the manifests.

### 8.2. Don't open unnecessary ports

Every port is a potential entry point. In a Dockerfile, the `EXPOSE` instruction doesn't publish a port by itself — it's more of a declaration of intent.

Good:

```dockerfile
EXPOSE 8080
```

But the actual publishing happens at runtime:

```bash
docker run -p 8080:8080 myapp
```

Practices:

- only declare the ports you actually need;
- don't run SSH inside the container;
- don't publish ports unnecessarily;
- for local development, bind to `127.0.0.1` rather than all interfaces, where appropriate.

### 8.3. Be careful with volumes and bind mounts

A bind mount can overwrite the contents of a directory inside the container. For example, if the image already has `/app`, and you mount the current directory there, the `/app` content from the image gets hidden.

A risky way to run:

```bash
docker run -v $(pwd):/app myapp
```

This is convenient for development, but it can break production behavior.

Practices:

- use named volumes for data;
- mount specific files for configs, not the whole project root;
- don't store mutable data inside the image;
- don't use a volume as a way to deliver secrets without controlling permissions;
- check that the volume doesn't require root ownership.

### 8.4. Don't bake environment-specific configs into the image

The image should be identical across dev, staging, and production. What should differ are the runtime settings: environment variables, secrets, config maps, mounted config files.

Bad:

```dockerfile
COPY config.prod.yml /app/config.yml
```

Better:

```dockerfile
COPY config.example.yml /app/config.example.yml
```

And pass the real production config in at runtime through the orchestrator.

### 8.5. Write logs to stdout/stderr

A containerized application shouldn't write its primary logs only to a file inside the container. Logs should go to stdout/stderr so Docker, Kubernetes, and external logging systems can collect them.

Bad:

```bash
myapp --log-file /var/log/myapp.log
```

Better:

```bash
myapp --log-format json
```

Or an application setting:

```
LOG_TO_STDOUT=true
```

Log files inside the container make rotation, collection, and diagnostics more complicated.

### 8.6. Limit CPU and memory at runtime

This isn't strictly a Dockerfile practice, but it's an important part of running containers in production. If a container has no limits set, it can eat up host memory or CPU and affect other services.

Docker:

```bash
docker run --cpus=2 --memory=512m myapp
```

Docker Compose:

```yaml
services:
  app:
    image: myapp:1.0.0
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 512M
        reservations:
          cpus: "1"
          memory: 256M
```

In Kubernetes, set `resources.requests` and `resources.limits`.

### 8.7. For Gunicorn, use a memory-backed worker temp directory

Gunicorn's heartbeat mechanism uses temporary files. If they sit on disk-backed storage, you can run into delays and stalls on operations like `os.fchmod`.

Practice:

```bash
gunicorn --worker-tmp-dir /dev/shm config.wsgi -b 0.0.0.0:8000
```

This is especially useful for Python web applications in containers.

---

## 9. Software supply chain, registries, signing, scanning, linting, testing, and CI/CD

Even a perfectly written Dockerfile doesn't fully protect you if images come from random sources, aren't signed, aren't scanned, are pushed only as `latest`, and CI has unnecessary privileges. So the practices around the image matter just as much as the Dockerfile itself.

### 9.1. Add metadata labels

Labels help clarify what an image is, who maintains it, where the source code lives, what version of the application it is, where the documentation is, and where to report security issues.

Example:

```dockerfile
LABEL org.opencontainers.image.title="myapp"
LABEL org.opencontainers.image.description="Example service"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.source="https://git.example.com/team/myapp"
LABEL org.opencontainers.image.vendor="Example Team"
LABEL org.opencontainers.image.licenses="MIT"
LABEL securitytxt="https://example.com/.well-known/security.txt"
```

For public images, a `security.txt` is useful: it tells researchers where to report security issues they've found.

### 9.2. Store images in a trusted registry

For production, it's better to use a private registry or a trusted enterprise image store. The public Docker Hub is convenient, but not all images there are maintained, updated, and scanned.

Practices:

- store internal images in a private registry;
- restrict push/pull permissions;
- enable scanning in the registry;
- don't pull random images from unknown authors into production;
- before using a third-party image, check its source, Dockerfile, signature, update frequency, and scan results.

### 9.3. Sign and verify images

An image tag can change. A registry can be compromised. MITM attacks and image substitution are real risks in this ecosystem.

Practice: sign your own images and verify the signature before use.

Docker Content Trust and Notary v1 used to be commonly mentioned in this context, but for new production workflows it's better to look at more modern mechanisms: Sigstore/Cosign, Notation/Notary Project, along with signature-verification policies in CI/CD, the registry, or a Kubernetes admission controller.

Example with Cosign:

```bash
IMAGE="registry.example.com/myapp@sha256:<digest>"
cosign sign "$IMAGE"
cosign verify "$IMAGE" \
  --certificate-identity="https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com"
```

Example with Notation:

```bash
IMAGE="registry.example.com/myapp@sha256:<digest>"
notation sign "$IMAGE"
notation verify "$IMAGE"
```

It's important not just to sign the image, but to build signature verification into the run or deployment pipeline. Otherwise the signature exists in isolation and doesn't actually protect production against image substitution.

### 9.4. Generate an SBOM and provenance

To control where an image comes from and what it's made of, it's important to understand not just that the image is signed, but exactly what went into it and how it was built.

An SBOM (Software Bill of Materials) is a list of the components inside an image: system packages, runtime dependencies, application libraries, and their versions. It helps you quickly figure out whether a new CVE affects a specific image, which dependencies need updating, and where a vulnerable component came from.

Provenance is information about the build's origin: which repository, commit, workflow, runner, and builder it came from, and with what parameters the image was built. This is especially useful when you need to prove that a production image was actually built from the right source code, rather than ending up in the registry manually or from an unknown pipeline.

Example with Docker Buildx:

```bash
docker buildx build \
  --sbom=true \
  --provenance=true \
  -t registry.example.com/myapp:${GIT_SHA} \
  --push \
  .
```

Practices:

- generate SBOMs for production images;
- store the SBOM and provenance alongside the image, or in an artifact system;
- use SPDX or CycloneDX formats if your security processes require them;
- link the SBOM/provenance to the image signature;
- verify provenance in CI/CD or in an admission policy if strict supply-chain protection is required.

An SBOM doesn't replace scanning, and provenance doesn't replace a signature. They complement each other: the signature answers "can this artifact be trusted," the SBOM answers "what's inside it," and provenance answers "how and where was it built."

### 9.5. Scan images for vulnerabilities, secrets, malware, and misconfiguration

Even a good Dockerfile can produce an image with a vulnerable base layer or dependency. That's why scanning should be part of CI/CD.

What to check:

- CVEs in system packages;
- CVEs in application dependencies;
- secrets;
- configuration mistakes;
- running as root;
- `ADD` without a checksum, or used where `COPY` would suffice;
- unnecessary open ports;
- suspicious files;
- malicious files, if that's a requirement.

Tools: Trivy, Snyk, Clair, Grype, Anchore, Dockle, and similar.

Example:

```bash
trivy image --scanners vuln,secret,misconfig myapp:1.2.3
```

Scanning should happen:

- locally before push;
- in CI after the build;
- in the registry after upload;
- periodically for already-published images, since new CVEs appear over time.

### 9.6. Use Dockerfile linters and Docker Build Checks

A linter catches typical mistakes before they reach production.

The best-known tool is `hadolint`.

Example:

```bash
hadolint Dockerfile
```

It can flag things like:

- missing a pinned tag;
- shell form of `CMD`/`ENTRYPOINT`;
- unnecessary sequential `RUN` instructions;
- missing package-manager cache cleanup;
- unsafe package-installation patterns.

In addition, use the built-in Docker Build Checks. These are checks that run during the build step and help catch Dockerfile mistakes before the image is even published.

Example:

```bash
docker buildx build --check .
```

Build Checks can flag things like:

- secrets used in `ARG` or `ENV`;
- attempting to copy a file excluded via `.dockerignore`;
- undefined `ARG` in `FROM`;
- outdated or ambiguous instruction syntax;
- shell form used where the JSON/exec form would be better.

`hadolint` and Docker Build Checks don't replace each other. It's better to use both: `hadolint` as an external linter with a broad rule set, and Build Checks as a built-in Docker/BuildKit check that understands the build context well.

Linting the Dockerfile should be a mandatory CI step.

### 9.7. Test the Dockerfile and the resulting image

A Dockerfile needs to be tested just like code.

A minimal set of checks:

```bash
docker build -t myapp:test .
docker run --rm myapp:test --version
docker run --rm myapp:test id
docker run --rm -p 8080:8080 myapp:test
```

What to verify:

- the container starts;
- the application responds on the health endpoint;
- the process doesn't run as root;
- the necessary files are present;
- there are no extra files or secrets;
- only the needed ports are open;
- SIGTERM correctly shuts down the application;
- the image passes the linter and scanner.

For formal checks, you can use container structure tests or similar tools.

### 9.8. Don't push only `latest` in CI/CD

CI pipelines often do this:

```bash
docker build -t myapp:latest .
docker push myapp:latest
```

This is bad: it's unclear which commit currently corresponds to `latest`, and you can't roll back properly.

Better to tag the image with several meaningful tags:

```bash
docker build \
  -t registry.example.com/myapp:1.2.3 \
  -t registry.example.com/myapp:git-${GIT_SHA} \
  .
```

A solid production deployment should reference an immutable tag or digest, not a floating `latest`.

### 9.9. Protect the Docker socket and the Docker TCP API

This isn't strictly a Dockerfile practice anymore, but it's an important practice around containers. `/var/run/docker.sock` grants almost root-level access to the host. If you mount the Docker socket into a container, the application inside it can control Docker on the host.

Dangerous:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Practices:

- don't mount the Docker socket unless absolutely necessary;
- if access to the Docker API is needed, restrict it through proxy/policy mechanisms;
- don't expose the Docker TCP API without TLS and authentication;
- monitor permissions on `/var/run/docker.sock`;
- don't run CI jobs with unnecessary privileges.

---

## 10. Final Dockerfile examples

Below are two examples that combine several practices into one working Dockerfile: a pinned base-image version, multi-stage build, BuildKit cache, non-root user, healthcheck, exec-form startup, minimal extra files, and write-protected code against the runtime user.

### 10.1. Example Dockerfile for a Python API

```dockerfile
# syntax=docker/dockerfile:1.8

FROM python:3.12.13-slim-bookworm AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./

RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --wheel-dir /wheels -r requirements.txt

FROM python:3.12.13-slim-bookworm AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN groupadd -r app && useradd -r -g app -d /nonexistent -s /usr/sbin/nologin app

COPY --from=builder /wheels /wheels
COPY requirements.txt ./
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels

COPY . /app
RUN chmod -R a-w /app

USER app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)" || exit 1

ENTRYPOINT ["gunicorn", "--worker-tmp-dir", "/dev/shm", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

What's included here:

- a pinned base-image version;
- `slim` rather than a full OS;
- multi-stage build;
- build dependencies don't end up in the runtime image;
- the pip cache doesn't end up in a layer;
- dependencies are installed before the code is copied;
- a non-root user;
- exec form of `ENTRYPOINT`;
- a healthcheck;
- Gunicorn's temp directory in `/dev/shm`;
- code that isn't writable by the runtime user.

### 10.2. Example Dockerfile for Node.js

```dockerfile
# syntax=docker/dockerfile:1.8

FROM node:24.16.0-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

FROM node:24.16.0-slim AS runtime
WORKDIR /app

ENV NODE_ENV=production

COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
COPY src/ ./src/

RUN chmod -R a-w /app

USER node

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "src/index.js"]
```

What's included here:

- a pinned Node.js version;
- dependencies installed separately from the source code;
- npm cache mount used;
- no `latest`;
- no root;
- no shell form for CMD;
- a healthcheck without installing curl;
- only the necessary directories are copied.

---

## Conclusion

A good Dockerfile isn't the shortest possible Dockerfile. A good Dockerfile is one that produces a small, clear, verifiable, and reproducible image. It has no random packages, no floating versions, no root process, no secrets in the layers, and no magic like `ADD` from a URL. It builds quickly, shuts down correctly, passes scanners, and behaves the same way in CI, staging, and production.

---

**Tags:** dockerfile, docker, best practice, kubernetes, images, containers, devops, devsecops, containerization

**Hubs:** DevOps, System Administration, Information Security, Kubernetes, Linux

**Author:** Seyd-Magomed Kuzgov ([@casssuzy](https://habr.com/ru/users/casssuzy/)), DevOps/DevSecOps engineer

**Original article:** [https://habr.com/ru/articles/1041784/](https://habr.com/ru/articles/1041784/)
