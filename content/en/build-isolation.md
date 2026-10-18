---
title: "Build isolation"
weight: 80
---

# Build isolation {#build-isolation}

Packages are only as trustworthy as the environment that produced them. The factory
builds in two phases, one shipped and one experimental.

## Phase A — debian container (shipped) {#phase-a-debian-container-shipped}

Builds run inside a pinned `debian:stable-slim` container rather than on the drifting
GitHub-runner host. This gives:

- a **controlled, known glibc floor** instead of whatever the runner ships;
- **no host-tool leakage** into the build — only the declared base plus pkgx-sourced
  dependencies are present;
- reproducible output across runs.

This is the isolation model in use for every published package today.

## Phase B — FROM-scratch pkgx-glibc toolchain (experimental) {#phase-b-from-scratch-pkgx-glibc-toolchain-experimental}

Phase A still links against the container's glibc. Phase B goes further: build against
pkgx's **own** glibc toolchain so packages are truly self-contained and run on a
literally-empty `FROM scratch` base with no system libc at all.

This is a [`bk`](https://github.com/go-pkgx/bk) change, currently **experimental /
proven feasible** — not yet the default:

- lives on branch `feat/pkgx-glibc-toolchain`;
- **off by default**, gated behind `BK_PKGX_LIBC=1`;
- design note:
  [`bk/docs/from-scratch-toolchain.md`](https://github.com/go-pkgx/bk/blob/feat/pkgx-glibc-toolchain/docs/from-scratch-toolchain.md).

See [Running pkgx packages on FROM scratch]({{< relref "from-scratch.md" >}}) for why a system-linked
package needs closure completion to run on scratch, and what Phase B removes.
