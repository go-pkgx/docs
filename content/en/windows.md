---
title: "Windows"
weight: 90
---

# pkgx Windows packages — feasibility study & proposal {#pkgx-windows-packages-feasibility-study-proposal}

**Date:** 2026-07-31 · pkgx has **no** Windows packages today (every
`dist.pkgx.dev/<project>/windows/*` returns 404). This studies whether that can
change, with a working cross-build proof, and proposes a path.

## TL;DR {#tldr}

Windows packages are **feasible and, for most packages, simpler than the
linux/darwin ones**. pkgx already ships the whole cross toolchain
(`llvm.org/mingw-w64`). The blockers are (1) an upstream decision to build them
and (2) a small runtime port of the client tools — not any deep technical wall.

## What was proven (on darwin/arm64, from this repo) {#what-was-proven-on-darwinarm64-from-this-repo}

| # | Finding | Evidence |
|---|---|---|
| 1 | **The cross-toolchain is already in the pantry.** `llvm.org/mingw-w64` (llvm-mingw) ships `x86_64-w64-mingw32-clang` **and** `aarch64-w64-mingw32-clang` (+ objdump/dlltool/…). So pkgx's own linux CI can cross-build Windows PE for both arches. | `pkgx +llvm.org/mingw-w64 -- x86_64-w64-mingw32-clang …` works |
| 2 | **Cross-building a PE works.** A C `hello.c` compiled to a valid `PE32+ executable (console) x86-64, for MS Windows`. | `file hello.exe` → PE32+ |
| 3 | **Leaf tools bundle nothing.** The PE imports only Windows-provided DLLs — `api-ms-win-crt-*` (Universal CRT, present on Windows 10+) and `KERNEL32.dll`. | `objdump -p` import table |
| 4 | **C++ can be fully self-contained.** `clang++ … -static` on a `<iostream>` program produces a PE whose only imports are Windows DLLs — **zero** bundled `libc++`/`libunwind`/`libwinpthread`. | `objdump -p hi-static.exe` |

## Why Windows packages are *simpler* than linux ones {#why-windows-packages-are-simpler-than-linux-ones}

The linux FROM-scratch work in this repo fought three things that **do not exist
on Windows**:

- **No glibc package.** Windows ships the C runtime (UCRT) as part of the OS.
  There is no `gnu.org/glibc` equivalent to mirror or to target.
- **No rpath/RUNPATH dance.** Windows resolves DLLs from the executable's own
  directory first, then `PATH`. A package just **colocates** any shared DLLs next
  to the `.exe` (or in a `lib/` dir put on `PATH`). No patchelf, no
  `--dynamic-linker`, no `$ORIGIN`.
- **No loader indirection.** There is no `ld-linux` to invoke explicitly; you
  exec the PE directly.

So the package shapes are:

- **leaf C tool** → a single `.exe`, no bundled libs.
- **C/C++ with static libs** (`-static`) → a single self-contained `.exe`.
- **shared-lib deps** → the `.exe` + the dependency DLLs colocated in the package.

## The two halves of "Windows support" {#the-two-halves-of-windows-support}

**(A) Producing packages** — an upstream (pkgxdev) concern. A pantry recipe would
cross-build with `llvm.org/mingw-w64` on the existing linux CI, e.g.

```yaml
platforms: [linux, darwin, windows]   # windows added
build:
  dependencies:
    llvm.org/mingw-w64: '*'
  script:
    # for autotools/cmake projects: --host=x86_64-w64-mingw32, CC=x86_64-w64-mingw32-clang, …
    - $CC $CFLAGS foo.c -o {{prefix}}/bin/foo.exe
```

Not every recipe ports cleanly (POSIX-only source, `configure` assuming a POSIX
host, projects with no Windows story), so this is **per-recipe, opt-in** —
exactly how darwin support already works. Pure-Go and Rust packages cross-compile
to Windows almost for free.

**(B) Consuming packages** — the go-pkgx client side, which we control. **Done and
proven on a real `windows-latest` runner** (go-pkgx v0.3.0):

- `bottle.Exec` was `syscall.Exec` (UNIX-only; on Windows it's a stub returning
  `EWINDOWS`). Split build-tagged: Windows spawns the target as a child with
  inherited stdio, waits, and exits with its code — a faithful `exec` analogue.
- `goos()` reports `"windows"`; `bin/foo` resolves to `foo.exe`.
- `run` on Windows: skip the ELF loader; set `PATH` (native separator) to the
  closure's `bin`/`lib` dirs and exec the `.exe`.

The CI job (`go-pkgx/pkgx`, `windows-latest`) builds `pkgx.exe`, fabricates a
tiny Windows package (`hello.exe`) served over HTTP, and asserts that `pkgx.exe`
**fetches and runs it** (`hello from pkgx package (win)`) **and propagates a
non-zero exit code** — both pass on real Windows. `mirror` already runs on
Windows unchanged (pure file+http), so a Windows box can host a mirror today.

Not yet done (follow-ups): `pkgm install` shims are `#!/bin/sh` (need `.cmd`);
the colocated-DLL case (a non-static shared dep) isn't exercised because no such
package exists yet.

## Proposal to pkgxdev {#proposal-to-pkgxdev}

1. Add `windows/x86-64` (+ `aarch64`) as first-class package slugs, built on the
   existing linux CI via `llvm.org/mingw-w64`, **opt-in per recipe** (`platforms`
   includes `windows`), starting with the packages that cross-build trivially
   (Go/Rust tools, single-file C tools).
2. Package relocatability on Windows = colocate DLLs (no rpath) — a much smaller
   brewkit change than the linux relocation logic.
3. go-pkgx provides a reference pure-Go client (`pkgx`/`pkgm`) that consumes
   Windows packages, and `mirror` to host them — so the ecosystem is testable
   independently of the reference Deno tooling.

## Honest limitations {#honest-limitations}

- Native-Windows consumption only helps once Windows packages exist; until then,
  **WSL2** remains the path for Windows users (linux binaries + linux packages).
- Packages whose upstreams have no Windows support won't get packages — same
  opt-in reality as darwin.
- go-pkgx's `run` closure-completion was built around ELF `DT_NEEDED`; the
  Windows equivalent (walking a PE import table for non-system DLLs) is a
  separate, smaller feature if/when shared-DLL packages appear.
