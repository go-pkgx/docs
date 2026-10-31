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

- ⛔ **`provides` is unreliable on this platform too, in both directions, and
  that one you can trip over.** The same defect loses the lane recipe's
  `provides`, and `provides` is what `pkgx -Q <command>`, `pkgs search` by
  command name and the `<TAB>` index are built from. Measured in the catalogue
  of 2026-10-31: `zlib.net` lists **no** command although its bottle ships
  `bin/minigzip.wasm`, so `minigzip` is published, runnable and unreachable by
  name; and `filippo.io/age` lists `age-keygen`, which that bottle does **not**
  contain, so the command resolves to a project that cannot supply it.

  Until bk#376 is fixed, **the bottle's own `bin/*.wasm` is the authority** for
  what a wasip1 project provides, not the catalogue. Both cases were found by
  running every published module — a catalogue consumer that only parses the
  catalogue cannot see either.

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

  ⛔ **And `--dir=/` does not make a relative path work.** Granting `/` does not
  put you in it: `prog data.json` can still fail to find `data.json` while
  `prog "$PWD/data.json"` succeeds. The factory's own C/C++ smoke tests pass
  absolute paths for exactly this reason. If a module cannot see a file you are
  sure you granted, try the absolute path before suspecting the module.
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

## What will not build for wasip1, and why {#what-will-not-build}

A `wasip1/wasm` bottle is only published after the module has been **run** under
wasmtime, so the lanes carry what works. This section is the other half: the
candidates that were tried and rejected, with the cause. It is here because a
documented "no, because X" is worth as much as a published bottle and saves the
next person the same hours.

Every cause below is a compiler or linker error from a real attempt at the
project's latest tag, on `GOOS=wasip1 GOARCH=wasm`. They group into five
families, and the families matter more than the names:

| family | what it needs that wasip1 has not | seen in |
| --- | --- | --- |
| **terminals and TTY ioctls** | `unix.Termios`, `unix.IoctlGetWinsize`, winsize ioctls, `isatty`, raw-mode echo control | `moby/term`, `reeflective/readline` (`GetSize`), `vbauerster/mpb`, `cheggaaa/pb` (`termutil.lockEcho`), `muesli/termenv` (`foregroundColor`), `fatih/color`, `logrus` (`isTerminal`), `bubbletea` v2 |
| **the clipboard** | a desktop selection | `atotto/clipboard` |
| **missing wasip1 backends** | a file-watching, SQL or socket implementation for this platform | `fsnotify.Watcher` (via `viper`), `modernc.org/libc` (so `modernc.org/sqlite`), `docker/go-connections` (`ForkLock`, `RawSockaddrUnix`) |
| **file metadata and syscalls** | `Stat_t` fields and calls WASI does not carry | `djherbis/atime` (`Stat_t.Atime`), `u-root/ls` (`Stat_t.Rdev`, `unix.Major`), `go-git` (`isSymlinkWindowsNonAdmin`), `syscall.Mkfifo` |
| **cgo** | a C toolchain at build time | `smacker/go-tree-sitter` |

Two that belong to no family and are worth knowing:

- **A compiler limit, not a platform limit.** `miller` fails with
  `function too big … exceeds 65536 blocks` in its *generated* parser. Nothing
  about WASI is involved; the wasm backend simply refuses a function that large.
- **`internal/robustio`** and **`azblob`**'s `Mmb` are build-constraint gaps in
  dependencies rather than anything a recipe can fix.

The practical rule that falls out: **a program that reads files or stdin and
writes stdout ports almost for free; a program that draws on a terminal does
not.** `fzf`, `less` and `nano` are terminal programs and should be expected to
fail, not attempted. Anything in Rust or C++ is a different lane entirely.

### A smoke test that only prints a version proves the module starts

Several lane entries smoke with `--version` or `-h`. That proves the module
loads, instantiates and exits 0 — which is more than nothing, and less than
working. Where a project takes real input the lanes pass it (`yq`, `gojq`, `jj`,
`tomljson`, `json2yaml`, `hcl2json` all run against a fixture), and that is the
standard to aim for when adding one.

### What wasi-libc does not have, for C and C++ {#wasi-libc-gaps}

The Go rejections above are a Go toolchain's view. C and C++ hit a different
wall, and these are the ones that cost real hours — each from a build that was
attempted, not from reading a header:

| the gap | how it shows up |
| --- | --- |
| **no `pipe()`** | links, then fails at run time. `xz` reaches `Error creating a pipe: Not supported` from `file_io.c`'s self-pipe — the hardest kind, because the build is green |
| **`signal.h` refuses to compile** | an `#error` unless `-D_WASI_EMULATED_SIGNAL` is set, and then `struct sigaction` is still *incomplete*: `sigaction`, `sigemptyset`, `sigfillset`, `sigprocmask`, `alarm` are undeclared |
| **declared but not defined** | `tmpfile`, `tmpnam`, `system` link-fail as "undefined symbol" although the headers declare them (deprecated). They have to be stubbed to fail the way a sandbox would |
| **defined as `ENOSYS`** | `fchmod`, `utime`, `chown` are already in `libc.a` returning `ENOSYS`, so defining your own gives a **duplicate symbol**. Rename yours with `-D` instead of redefining |
| **`chown`/`fchown` undeclared** | while `chown` is simultaneously present in `libc.a` as the row above — declaration and definition disagree |
| **no `dlopen`** | any plugin or dynamic-module feature has to be configured off; it is why `lua` builds its generic `ansi` platform rather than `posix` |
| **`setjmp`/`longjmp` need a flag** | `-mllvm -wasm-enable-sjlj -mllvm -wasm-use-legacy-eh=false`, per WebAssembly/wasi-sdk's `SetjmpLongjmp.md`. The standard instructions, not the legacy ones |
| **a 64 KiB default stack** | deep recursion *traps* rather than erroring. `lz4 -d` needed `-Wl,-z,stack-size=8388608` |

The pattern across all of them: **a green build says very little.** Three of the
eight above pass compilation and linking and fail only when the program runs,
which is why every bottle on this platform is run under wasmtime before it is
published.

### ⛔ A bottle's bytes are not stable across a toolchain bump {#bytes-and-toolchain}

The C/C++ lane resolves its toolchain — `llvm.org`, `wasi-libc`,
`compiler-rt.llvm.org/wasi`, `libcxx.llvm.org/wasi` — **at build time**, and
pins nothing. So rebuilding an unchanged recipe at an unchanged project version
can produce a different module, and on 2026-10-31 it did:
`github.com/simdjson/simdjson` v5.0.5 was republished with different bytes (the
same 432 531 of them) and identical behaviour.

Measured at the same time, and worth knowing because it bounds the problem:
`zlib.net`, `lua.org` and `sqlite.org` were rebuilt by the same runs and came
out **byte-identical**. simdjson is the only C++ project of the four — the only
one linking the floating `libcxx`.

Tracked in [go-pkgx/packages#393](https://github.com/go-pkgx/packages/issues/393),
together with the separate finding that the publish step replaced an existing
`(project, version, platform)` instead of skipping it. Until both are closed,
**a version does not pin bytes on this platform**; a `bk lock` pins the version,
and that is a weaker guarantee than it sounds.

### A tag that is not `v<version>`

The lanes clone `v<version>`. A project tagging `2023.1.7` or `jp/v1.0.0`
therefore cannot be expressed yet — `staticcheck`, `jmespath/jp` and
`golang/tools` (for `goimports`) are waiting on a `tag:` field in the matrix.

## Bibliography

- WASI preview1's `docs.md` on the [`wasi-0.1`
  branch](https://github.com/WebAssembly/WASI/blob/wasi-0.1/legacy/preview1/docs.md)
  — the type layouts and the errno ordinals.
- wasi-libc's `__wasilibc_real.c` — the import signatures as actually declared.
  `docs.md` writes `path: string`; only the real declarations show it lowered
  to a pointer and a length.
- wasi-libc's `wasip1.h` — `__WASI_ERRNO_NOSYS (UINT16_C(52))`, which is how to
  check an errno ordinal was counted rather than guessed.
