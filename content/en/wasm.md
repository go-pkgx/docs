---
title: "wasip1/wasm"
weight: 85
---

# Packages that run in a WebAssembly runtime {#packages-that-run-in-a-webassembly-runtime}

`wasip1/wasm` is a platform of the factory like `linux/x86-64` or
`darwin/aarch64`: the same recipes, the same `bk publish`, the same registry at
`oci://ghcr.io/go-pkgx/packages`, and the same catalogue. What differs is the
one property that changes everything downstream.

## A wasip1 module is statically linked, so there is no closure

On every other platform a bottle is a node in a dependency graph. `pkgx` walks
it, installs the transitive closure, and sets up the environment the dynamic
loader needs — `DT_NEEDED` on ELF, `@rpath` on Mach-O.

A `wasm32-wasip1` module has no dynamic loader. Its libraries are linked into
the module, so a single `.wasm` file is the whole program. This is not an
optimisation of the install path; it is the absence of the thing the install
path exists for.

Two consequences follow, and the second is a trap:

- **One file is enough.** Fetching one bottle gives you something you can run.
  There is nothing else to resolve.
- **Do not resolve a `wasip1/wasm` closure.** A consumer that walks the deps
  before running the module will fail on packages it does not need — and, right
  now, on packages the module does not even contain. `lua.org`'s catalogue entry
  on this platform names `gnu.org/readline`, which is not published for
  `wasip1/wasm` at all; and the wasm recipe does not use readline either. It
  builds Lua on the generic `ansi` platform with no line editor, and its only
  declared dependency is `gnu.org/make`, at build time.

  That entry is a **defect, not a build-time fact**: `bk catalog` reads the
  dependencies from the *pantry host* recipe when a platform lane has a recipe
  of its own, so the wasm entry inherits the host's `deps`. Tracked in
  [go-pkgx/bk#376](https://github.com/go-pkgx/bk/issues/376). Nothing breaks at
  run time, because nothing needs resolving — which is exactly why it went
  unnoticed.

## Running one

The reference runtime is [wasmtime](https://wasmtime.dev). Every bottle is run
under it before it is published — the factory's `wasip1.yml` (Go lane) and
`wasip1-c.yml` (C/C++ lane) both do this, and a module that will not run is
not published.

The invocation they use is the one to copy:

```sh
wasmtime run --dir=/ --env PWD="$PWD" ./sqlite3.wasm :memory: "select sqlite_version();"
```

Both flags are load-bearing:

- `--dir=/` grants a filesystem. **WASI has no ambient one**: a module with no
  preopened directory cannot open any path, and the failure looks like a broken
  program rather than a missing grant.
- `--env PWD="$PWD"` tells Go's `wasip1` runtime which preopen the working
  directory is in — the same thing `$GOROOT/lib/wasm/go_wasip1_wasm_exec` does.
  Without it a relative path resolves against nothing.

The arguments come after the module with **no `--` separator**.

## Finding them

The catalogue is published per platform, so ask for this one:

```sh
pkgs catalog update --platform wasip1/wasm
pkgs ls
pkgs ls --tree lua.org
```

As with every other platform, `pkgs` reads the local catalogue and nothing else
— `<TAB>` completion included — so this works offline once the catalogue is
fetched. See [Finding packages]({{< relref "finding-packages.md" >}}).

## Running one in a browser

The modules are the same bottles; a browser is just another WASI host. The
landing page carries a playground that runs them client-side, at
[/playground/](https://go-pkgx.github.io/playground/).

Two details of how it is built are worth knowing if you build something
similar:

- **`ghcr.io` sends no CORS headers.** A page cannot fetch a bottle from the
  registry; every browser refuses it, and no test on the page catches that
  because the page's own tests do not cross an origin. The modules have to be
  served from the page's own origin, which means a build step copies them
  there.
- **An unimplemented syscall must not return 0.** A host that answers "success"
  to a call it never implemented produces a run that looks exactly like one
  that worked. The playground's host returns `ENOSYS` (errno **52** in
  preview1) and names the call on the page.

That second point is why the playground's host is checked by running the **same
modules under wasmtime** and requiring the same stdout, the same stderr and the
same exit status. It has already caught one difference worth repeating here:
`sqlite3` printed a box-drawing table where wasmtime printed `3.53.4`, because
wasi-libc's `isatty()` answers from the filetype and rights that
`fd_fdstat_get` reports, and a redirected stdout is `UNKNOWN` with `FD_WRITE`
alone. A host that reports a terminal where the reference reports a pipe
changes the output of programs that format for humans — silently, and only for
some of them.

## Bibliography

- WASI preview1's `docs.md` on the [`wasi-0.1`
  branch](https://github.com/WebAssembly/WASI/blob/wasi-0.1/legacy/preview1/docs.md)
  — the type layouts and the errno ordinals.
- wasi-libc's `__wasilibc_real.c` — the import signatures as actually declared.
  `docs.md` writes `path: string`; only the real declarations show it lowered
  to a pointer and a length.
- wasi-libc's `wasip1.h` — `__WASI_ERRNO_NOSYS (UINT16_C(52))`, which is how to
  check an errno ordinal was counted rather than guessed.
