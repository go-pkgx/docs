---
title: "Home"
weight: 10
---

# go-pkgx {#go-pkgx}

A **dependency-free, pure-Go** (`CGO_ENABLED=0`) package manager for
[pkgx](https://pkgx.sh) packages — a single static binary that installs **and
runs** packages on a `FROM scratch` image, with no runtime dependencies of its
own.

The reference `pkgm` is a Deno/TypeScript script that shells out to `pkgx`,
`deno`, `curl`, `openssl`, `zlib` and `xz` — roughly **515 MB** of runtime
closure just to install a package. go-pkgx replaces all of it with one static
~9 MB binary.

## ⛔ Three commands, and why `pkgx` and `pkgs` are two of them {#three-commands}

| | |
| --- | --- |
| `pkgm` | **installs** a package, into a store you keep |
| `pkgx` | **runs** one, on the fly, without installing it |
| `pkgs` | **answers about** one: `ls`, `search`, `why`, `catalog`, `store`, `browse`, `compat`, `env`, `completion` |

The last two ship from one repository and share one resolver. They are
separate commands because of a grammar, not a preference.

Upstream pkgx has no verbs at all: the first argument that is not a flag is
a **package**, and pkgx runs it. This implementation had added nine verbs in
that position, and each took a name away from the registry. Two took names
that exist — measured against the published linux/x86-64 catalogue on
2026-10-18, **`gnu.org/coreutils` provides both `ls` and `env`** — so `pkgx
ls` listed a catalogue where upstream runs GNU ls, and there was no way to
ask for the other one.

```console
$ pkgx ls --version          # GNU coreutils, as upstream does
ls (GNU coreutils) 9.11
$ pkgs ls gnu.org            # what is available under a node
$ pkgx -Q ls                 # which project provides a command
gnu.org/coreutils
```

A reserved word could not have fixed it: `vercel.com/pkg` provides the
command `pkg`, and any bare word can become a package tomorrow, because the
point of the first position is that the **registry** decides what is in it.
Only a token the grammar already reserves is safe — a flag, which is what
upstream chose for `pkgx -Q`, or a different binary.

## Install {#install}

**Linux / macOS** — one line, naming the release you want:

```sh
curl -fsSL https://go-pkgx.github.io/install.sh | sh -s -- pkgm v0.5.1
```

**Windows** (PowerShell) — `irm | iex` passes no arguments, so the version goes
in the environment:

```powershell
$env:PKGM_VERSION='v0.5.1'; irm https://go-pkgx.github.io/install.ps1 | iex
```

The installer downloads the static binary for your os/arch from that
[release](https://github.com/go-pkgx/pkgm/releases), verifies it against the
release `SHA256SUMS`, and puts `pkgm` on your `PATH` (`$HOME/.local/bin`, or
`%LOCALAPPDATA%\Programs\go-pkgx` on Windows; set `PKGM_INSTALL` to override on
Unix).

⛔ **`SHA256SUMS` is a checksum of itself.** It says the binaries match a
manifest published beside them, and whoever replaced the binaries would
replace the manifest. The half that cannot be forged from the release page is
GitHub's **provenance attestation** — signed through Sigstore, saying that the
repository's own release workflow built that file from that commit:

```sh
gh attestation verify -R go-pkgx/pkgm pkgm-linux-amd64
```

Every file of every `pkgx`, `pkgs`, `bk`, `pkgm` and `mirror` release
published from 2026-10-20 carries one. Earlier releases carry none, and
`gh attestation verify` answers `HTTP 404` for those rather than failing
closed, so a 404 on an older binary means *unattested*, not *tampered with*.

The version is named on purpose: this line copied today and the same line
copied in six months install the same bytes, and a bad release does not reach
everyone who happens to install that hour. To track releases instead, say so —
`sh -s -- pkgm latest`, or `PKGM_VERSION=latest`, which is also what a bare
`| sh` has always done. Re-running is the updater; it skips the download when
the target version is already installed.

To install `pkgx` (the runtime) or `mirror` instead, name it:
`sh -s -- pkgx v0.26.0`, `sh -s -- mirror v0.3.12`.

**Go users**:

```sh
go install github.com/go-pkgx/pkgm@latest
```

Once installed, `pkgm install lz4.org` verifies each bottle against the signed
registry by default.

## Usage {#usage}

```
pkgm install|i    <pkg>[@version] ...   install to /usr/local (root) or ~/.local
pkgm uninstall|rm <pkg> ...             remove an installation
pkgm shim|stub    <pkg> ...             create a shim in <prefix>/bin
pkgm list|ls                            list what's installed
pkgm outdated                           list outdated installations
pkgm update|up|upgrade                  update installations to latest
pkgm pin          <pkg>@version ...     install pinned to an exact version
pkgm run|x        <pkg> [-- args...]    run a pkg (works FROM scratch)

flags: -h/--help  -v/--version  -p/--pin  -P/--prefix DIR  -s/--from-scratch
env:   PKGX_DIR     package store (default: ~/.pkgx)
       PKGX_DIST    package source (default: oci://ghcr.io/go-pkgx/packages, signed)
       PKGX_VERIFY  verify signatures, fail-closed (default: on)
       PKGM_PREFIX  default install prefix (ideal for FROM scratch)
```

The `install`/`uninstall`/`shim`/`list`/`outdated`/`update`/`pin` command
surface and the `~/.local` vs `/usr/local` prefix logic mirror the reference
`pkgm`, so it is a drop-in replacement.

## FROM scratch {#from-scratch}

A scratch image whose only file is the `pkgm` binary can install and run real
packages, with no system libc:

```dockerfile
FROM scratch
COPY pkgm /pkgm
ENV PKGX_DIR=/pkgx
ENTRYPOINT ["/pkgm"]
```

```sh
$ docker run --rm pkgm-scratch run gnu.org/bash -- --version
GNU bash, version 5.3.0(1)-release (aarch64-unknown-linux-gnu)
```

See [FROM scratch]({{< relref "from-scratch.md" >}}) for how the closure is resolved, the
conformance matrix, and the pantry-wide audit.

## Environments {#environments}

`pkgx` also carries `pkge`, a module system: named environments declared in
HCL2, `load`/`unload`/`purge` that are exact rather than incremental, and a
converter that reads a site's existing Lmod and Environment Modules files.

```console
$ eval "$(pkgs env init)"
$ pkge load cfd
```

See [Environments and HPC]({{< relref "environments.md" >}}).
