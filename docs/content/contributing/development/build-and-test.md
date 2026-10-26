---
# SPDX-FileCopyrightText: 2026 Authelia
#
# SPDX-License-Identifier: Apache-2.0

title: "Building and Testing"
description: "A guide to building and testing Authelia from source covering unit tests, integration suites, and manually building the Go binary with the React frontend."
summary: "This section covers the build process and how to perform tests in development."
date: 2024-03-14T06:00:14+11:00
draft: false
images: []
weight: 240
toc: true
aliases:
  - /docs/contributing/build-and-dev.html
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

**Authelia** is built a [React] frontend user portal bundled in a [Go] application which acts as a basic web server for
the [React] assets and a dedicated API.

The GitHub repository comes with a CLI dedicated to developers called
[authelia-scripts](../../reference/cli/authelia-scripts/authelia-scripts.md) which can be setup by looking at
the [authelia-scripts reference guide](../../reference/cli/authelia-scripts/authelia-scripts.md).

In order to build and contribute to **Authelia**, you need to make sure that you have looked at the
[Environment](environment.md) guide to configure your development environment.

## Get started

In order to ease development, **Authelia** uses the concept of [suites] to run Authelia from source code so that your
patches are included. This is a kind of virtual environment running **Authelia** in a complete ecosystem
(LDAP, Redis, SQL server). Note that **Authelia** is hot-reloaded in the environment so that your patches are instantly
included.

The next command starts the suite called _Standalone_:

```bash
authelia-scripts suites setup Standalone
```

Most of the suites are using docker compose to bootstrap the environment. Therefore, you can check the logs of all
application by running the following command on the component you want to monitor.

```bash
docker logs authelia-authelia-backend-1 -f
```

Then, edit the code and observe how **Authelia** is automatically reloaded.

### Unit tests

To run the unit tests, run:

```bash
authelia-scripts unittest
```

### Integration tests

Integration tests are located under the `internal/suites` directory and are based on go-rod. A suite is a combination
of environment and tests. Executing a suite therefore means starting the environment, running the tests and tearing down
the environment. Each step can be run independently:

```bash {title="List the Available Suites"}
authelia-scripts suites list
```

```bash
Standalone
DuoPush
LDAP
Traefik
```

```bash {title="Start the environment of the Standalone suite"}
authelia-scripts suites setup Standalone
```

```bash {title="Run the tests related to the currently running suite"}
authelia-scripts suites test
```

```bash {title="Tear down the environment of the currently running suite"}
authelia-scripts suites teardown Standalone
```

In order to test all suites (approx 30 minutes), you need to make sure there is no currently running suite and then you
should run:

```bash
authelia-scripts suites test
```

Also, you don't need to start the suite before testing it. Given you're not running any suite, just use the following
command to test the _Standalone_ suite.

```bash
authelia-scripts suites test Standalone
```

The suite will be spawned, tests will be run and then the suite will be torn down automatically.

## Manually Building

### Binary

If you want to manually build the binary from source you will require the open source software described in the
[Development Environment](./environment.md#setup) documentation. Then you can follow the below steps on Linux (you may
have to adapt them on other systems).

Clone the Repository:

```bash
git clone https://github.com/authelia/authelia.git
```

Download the Dependencies:

```bash
cd authelia && go mod download
cd web && pnpm install
cd ..
```

Build the Web Frontend:

```bash
cd web && pnpm build
cd ..
cp -r api internal/server/public_html/api
```

Build the Binary (with debug symbols):

```bash
CGO_ENABLED=1 CGO_CPPFLAGS="-D_FORTIFY_SOURCE=2 -fstack-protector-strong" CGO_LDFLAGS="-Wl,-z,relro,-z,now" \
go build -ldflags "-linkmode=external" -trimpath -buildmode=pie -o authelia ./cmd/authelia
```

Build the Binary (without debug symbols):

```bash
CGO_ENABLED=1 CGO_CPPFLAGS="-D_FORTIFY_SOURCE=2 -fstack-protector-strong" CGO_LDFLAGS="-Wl,-z,relro,-z,now" \
go build -ldflags "-linkmode=external -s -w" -trimpath -buildmode=pie -o authelia ./cmd/authelia
```

## Container Image

The official container image is assembled by the [Dockerfile] in the root of the repository from two inputs:

1. The binary built by [authelia-scripts] (or by [GoReleaser] for a release), staged in a directory named after the
   target platform, e.g. `authelia-linux-amd64/authelia`.
2. The `authelia/base` image, which is assembled by the [baseimage] repository and referenced by both a tag and a
   digest.

The image for the platform of the host you are building on can therefore be assembled with:

```bash {title="Stage the binary"}
mkdir -p "authelia-linux-$(go env GOOS)-$(go env GOARCH)"
cp dist/authelia "authelia-linux-$(go env GOOS)-$(go env GOARCH)/"
```

```bash {title="Assemble the image"}
docker buildx build \
  --build-arg TAG=latest \
  --build-arg SHA="$(docker buildx imagetools inspect authelia/base:latest --format '{{.Manifest.Digest}}' | cut -d: -f2)" \
  --tag authelia/authelia:local .
```

The `TAG` and `SHA` build arguments are the tag and the digest of the `authelia/base` image index which the
`Dockerfile` resolves for the target platform.

The release pipeline assembles the image for every supported platform at once with
`authelia-scripts docker build` and publishes the manifest list with `authelia-scripts docker push-manifest`.

### LoongArch

The `linux/loong64` image is published like the other platforms. Its base image is assembled from the [Debian]
archive by `Dockerfile.loong64` in the [baseimage] repository because [Ubuntu] publishes no loong64 archive, and the
`linux/loong64` target of [GoReleaser] is built with the `loongarch64-linux-gnu-gcc` cross compiler instead of one
of the musl compilers, so the `-musl` archive is not published for it.

[suites]: ./integration-suites.md
[Dockerfile]: https://github.com/authelia/authelia/blob/master/Dockerfile
[GoReleaser]: https://goreleaser.com/
[authelia-scripts]: ../../reference/cli/authelia-scripts/authelia-scripts.md
[baseimage]: https://github.com/authelia/baseimage
[Ubuntu]: https://ubuntu.com/
[Debian]: https://www.debian.org/
[React]: https://reactjs.org/
[go]: https://go.dev/dl/
[Node.js]: https://nodejs.org/en/download/
[Docker]: https://docs.docker.com/get-docker/
[Docker Compose]: https://docs.docker.com/compose/install/
[golangci-lint]: https://golangci-lint.run/usage/install/
