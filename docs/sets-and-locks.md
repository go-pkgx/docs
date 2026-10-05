# Naming a tree, and pinning it

A recipe describes one package. Two things were missing above it: a way to
name a **group** of packages, and a way to say **which** group — because the
same names resolve differently tomorrow.

## A set is an ordinary recipe

A recipe carrying `members` is a set:

```hcl
# go-pkgx/base-toolchain
members = {
  # clang, lld, compiler-rt — the compiler itself
  "llvm.org" = "*"

  # libc, crt objects and the dynamic loader
  "gnu.org/glibc" = "*"
}
```

Anything that takes a project name takes a set: `bk closure
go-pkgx/base-toolchain`, `bk factory --recipes go-pkgx/base-toolchain`. A set
is expanded **at the root**, before the dependency walk sees it, so nothing
downstream has to learn the word.

The three systems that solved this agree on the shape, and Nix's answer is the
one taken:

| | |
| --- | --- |
| **Nix** | a tree *is* a derivation. `buildEnv` takes a list of packages and its output is a tree of symlinks. No new kind of object. |
| **Guix** | a manifest names packages; a **profile** is the tree it makes. |
| **Spack** | `spack.yaml` is the abstract set, `spack.lock` the concrete one. |

Because a set is an ordinary recipe, it is signed, attested, published and
installed by everything that already does those things for a package — rather
than a second kind of artefact to keep in step.

A set is **unordered**. All three of those manifests are: the order comes out
of resolution, not out of the file.

## A manifest alone is not reproducible

Guix's manual says it plainly: *"to reproduce a profile bit-for-bit, manifests
alone might not be enough"*. The same names resolve differently against a
different package set, so a manifest needs the channel revisions beside it.

**And it matters more here than in Guix.** A Guix manifest resolves
deterministically once the channel revision is pinned, because the version is
*in* the checkout. A pkgx recipe's `versions:` asks GitHub for tags **at
resolve time**, so pinning the pantry commit is not enough: the same pantry
resolves `gnu.org/binutils` to whatever the newest tag is the day you ask.

`bk lock` writes the answer down:

```console
$ bk lock --platform linux/x86-64 -o base-toolchain.lock.hcl go-pkgx/base-toolchain
set: go-pkgx/base-toolchain names 25 member(s)
lock: 41 project(s) pinned → base-toolchain.lock.hcl
```

```hcl
lockfile_version = 1
bk               = "v0.7.0"
platform         = "linux/x86-64"
generated        = "2026-10-05T09:10:39Z"
pantry           = "fd646990024e22f5752578cfdcfe2518b20c0797"
overlay          = "f8f6dcfd7b17410a6de780bdd58c81fbbb38d1b3"
roots            = ["go-pkgx/base-toolchain"]

locked = {
  "curl.se"      = { version = "8.17.0", spec = "sha256:5284d597c180…" }
  "gnu.org/bash" = { version = "5.3",    spec = "sha256:…" }
}
```

The **platform is in the header because it changes the answer**: the same set
locks to 41 projects on `linux/x86-64` and 40 on `darwin/aarch64`
(`github.com/besser82/libxcrypt` is linux-only). A lock that did not say which
platform it was taken on would be read as the other one's.

## `spec` — because a version is not enough either

A version pins what upstream served. It does not pin the **recipe**: a build
script edited in the pantry resolves to the same version and produces a
different package.

Spack's packaging guide names what a spec hash covers: *"`build`, `link`, and
`run` dependencies all affect the hash of Spack packages (along with `sha256`
sums of patches and archives used to build the package, and a **canonical hash
of the `package.py` recipes**)"*. The last clause is the one copied here.

`spec` is a Merkle hash over the platform, the project, the resolved version,
the **parsed** recipe and the spec hashes of its dependencies. Over the parsed
value and not the file, so reformatting a recipe or rewriting its comments does
not move it and changing what it says does.

## Checking a lock later

Cargo's `--locked` and npm's `ci` exist because a lock nobody verifies is a
lock nobody can rely on. `bk lock --check` re-resolves and says what moved:

```console
$ bk lock --check base-toolchain.lock.hcl
lock: linux/x86-64 · 2026-10-05T09:10:39Z · 41 pinned, 2 hour(s) old
lock: pantry revision differs: fd64699… → 9ab12cd…
  gnu.org/binutils                   2.47 → 2.48
  zlib.net                           spec 6a2d499db9ab → spec 90fae00e667c — same version, so something it is built FROM changed
lock: 2 of 41 moved
```

| exit | |
| --- | --- |
| `0` | nothing moved |
| `1` | something moved |
| `2` | the check could not be made at all |

**The second line of that diff is the one a version-only lock cannot
produce.** It is what the spec hash is for.

Two deliberate kindnesses to whoever has to run this in CI:

- a **revision that moved is reported and is not fatal on its own**. A pantry
  can move without moving any answer, and calling that drift would make the
  check cry wolf until nobody ran it.
- an **unresolved project is kept apart from drift**, for the mirror-image
  reason: a registry outage must not read as a pantry that changed.

`--check` takes its roots, its platform and the revisions it expects **from the
file**, and refuses a project name beside it — naming them twice would let the
two disagree, and the answer would be about a different question from the one
the file asks.

## What a lock does not pin

The **output bytes**. The spec hash covers the inputs, as Nix's derivation hash
and Spack's spec hash do; an output digest is a different promise and is not
made here. Every lock says so in its own header rather than letting a reader
assume a guarantee that is not there.
