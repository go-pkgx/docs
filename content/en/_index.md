---
title: "Home"
weight: 10
---

# go-zpkg {#go-zpkg}

A **dependency-free, pure-Go** (`CGO_ENABLED=0`) package manager — a single
static binary that installs **and runs** packages on a `FROM scratch` image,
with no runtime dependencies of its own: one static ~9 MB binary, no libc,
no interpreter.

It descends from pkgx — see [Filiation]({{< relref "filiation.md" >}}) for
what was taken and what was rewritten — but **it is no longer compatible with
it**, and no longer tries to be: the pantry, the recipe format and the
distribution are ours.

Four things it does, all four ordinary in Nix, Guix and Spack: build options
on a specification, two builds of one version in one store, a hashed identity
for a build, and a distribution that can be verified. See
[Addressing a build]({{< relref "addressing-a-build.md" >}}) for how a build
gets an identity and how you name the one you want.

The `z` is ZFS's: the last one you need.

## ⛔ Three commands, and why `zpkgx` and `zpkgs` are two of them {#three-commands}

| | |
| --- | --- |
| `zpkgm` | **installs** a package, into a store you keep |
| `zpkgx` | **runs** one, on the fly, without installing it |
| `zpkgs` | **answers about** one: `ls`, `search`, `why`, `catalog`, `store`, `browse`, `compat`, `env`, `completion` |

The last two ship from one repository and share one resolver. They are
separate commands because of a grammar, not a preference.

Upstream zpkgx has no verbs at all: the first argument that is not a flag is
a **package**, and zpkgx runs it. This implementation had added nine verbs in
that position, and each took a name away from the registry. Two took names
that exist — measured against the published linux/x86-64 catalogue on
2026-10-18, **`gnu.org/coreutils` provides both `ls` and `env`** — so `zpkgx
ls` listed a catalogue where upstream runs GNU ls, and there was no way to
ask for the other one.

```console
$ zpkgx ls --version          # GNU coreutils, as upstream does
ls (GNU coreutils) 9.11
$ zpkgs ls gnu.org            # what is available under a node
$ zpkgx -Q ls                 # which project provides a command
gnu.org/coreutils
```

A reserved word could not have fixed it: `vercel.com/pkg` provides the
command `pkg`, and any bare word can become a package tomorrow, because the
point of the first position is that the **registry** decides what is in it.
Only a token the grammar already reserves is safe — a flag, which is what
upstream chose for `zpkgx -Q`, or a different binary.

## Install {#install}

**Linux / macOS** — one line, naming the release you want:

```sh
curl -fsSL https://go-zpkg.github.io/install.sh | sh -s -- zpkgm v0.9.0
```

**Windows** (PowerShell) — `irm | iex` passes no arguments, so the version goes
in the environment:

```powershell
$env:ZPKGM_VERSION='v0.9.0'; irm https://go-zpkg.github.io/install.ps1 | iex
```

The installer downloads the static binary for your os/arch from that
[release](https://github.com/go-zpkg/zpkgm/releases), verifies it against the
release `SHA256SUMS`, and puts `zpkgm` on your `PATH` (`$HOME/.local/bin`, or
`%LOCALAPPDATA%\Programs\go-zpkg` on Windows; set `ZPKGM_INSTALL` to override on
Unix).

⛔ **`SHA256SUMS` is a checksum of itself.** It says the binaries match a
manifest published beside them, and whoever replaced the binaries would
replace the manifest. The half that cannot be forged from the release page is
GitHub's **provenance attestation** — signed through Sigstore, saying that the
repository's own release workflow built that file from that commit:

```sh
gh attestation verify -R go-zpkg/zpkgm zpkgm-linux-amd64
```

Every file of every `zpkgx`, `zpkgs`, `zbk`, `zpkgm` and `mirror` release
published from 2026-10-20 carries one. Earlier releases carry none, and
`gh attestation verify` answers `HTTP 404` for those rather than failing
closed, so a 404 on an older binary means *unattested*, not *tampered with*.

The version is named on purpose: this line copied today and the same line
copied in six months install the same bytes, and a bad release does not reach
everyone who happens to install that hour. To track releases instead, say so —
`sh -s -- zpkgm latest`, or `ZPKGM_VERSION=latest`, which is also what a bare
`| sh` has always done. Re-running is the updater; it skips the download when
the target version is already installed.

To install `zpkgx` (the runtime) or `mirror` instead, name it:
`sh -s -- zpkgx v0.32.0`, `sh -s -- mirror v0.4.0`.

**Go users**:

```sh
go install github.com/go-zpkg/zpkgm@latest
```

Once installed, `zpkgm install lz4.org` verifies each bottle against the signed
registry by default.

## Usage {#usage}

```
zpkgm install|i    <pkg>[@version] ...   install to /usr/local (root) or ~/.local
zpkgm uninstall|rm <pkg> ...             remove an installation
zpkgm shim|stub    <pkg> ...             create a shim in <prefix>/bin
zpkgm list|ls                            list what's installed
zpkgm outdated                           list outdated installations
zpkgm update|up|upgrade                  update installations to latest
zpkgm pin          <pkg>@version ...     install pinned to an exact version
zpkgm run|x        <pkg> [-- args...]    run a pkg (works FROM scratch)

flags: -h/--help  -v/--version  -p/--pin  -P/--prefix DIR  -s/--from-scratch
env:   ZPKG_DIR     package store (default: ~/.zpkg)
       ZPKG_DIST    package source (default: oci://ghcr.io/go-pkgx/packages, signed)
       ZPKG_VERIFY  verify signatures, fail-closed (default: on)
       PKGM_PREFIX  default install prefix (ideal for FROM scratch)
```

The command surface is `install`/`uninstall`/`shim`/`list`/`outdated`/
`update`/`pin`, with `~/.local` for an unprivileged install and `/usr/local`
otherwise.


## FROM scratch {#from-scratch}

A scratch image whose only file is the `zpkgm` binary can install and run real
packages, with no system libc:

```dockerfile
FROM scratch
COPY zpkgm /zpkgm
ENV ZPKG_DIR=/zpkgx
ENTRYPOINT ["/zpkgm"]
```

```sh
$ docker run --rm zpkgm-scratch run gnu.org/bash -- --version
GNU bash, version 5.3.0(1)-release (aarch64-unknown-linux-gnu)
```

See [FROM scratch]({{< relref "from-scratch.md" >}}) for how the closure is resolved, the
conformance matrix, and the pantry-wide audit.

## Environments {#environments}

`zpkgx` also carries `zpkge`, a module system: named environments declared in
HCL2, `load`/`unload`/`purge` that are exact rather than incremental, and a
converter that reads a site's existing Lmod and Environment Modules files.

```console
$ eval "$(zpkgs env init)"
$ zpkge load cfd
```

See [Environments and HPC]({{< relref "environments.md" >}}).

## WebAssembly {#webassembly}

`wasip1/wasm` is a platform of the factory like any other, with one property
that changes everything downstream: a `wasm32-wasip1` module is **statically
linked**, so one file is the whole program and there is no closure to resolve.
Do not try to resolve one: a `wasip1/wasm` catalogue entry can name a package
that is not published for that platform and that the module does not contain.

See [wasip1/wasm]({{< relref "wasm.md" >}}).
