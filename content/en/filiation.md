---
title: "Filiation: pkgx, and what pkgx cannot be"
weight: 15
---

# Filiation {#filiation}

go-zpkg descends from [pkgx](https://pkgx.sh). It reads pkgx's pantry, speaks
pkgx's bottle protocol, and resolves pkgx's dependency ranges. Everything it
installs is a pkgx package, and that is the point: the pantry is thousands of
recipes that work, and nothing here would exist without them.

It is not a reimplementation of pkgx, and the difference is not a matter of
ambition. Four things this needs are **unreachable from pkgx's design**, not
missing from its implementation, and all four are ordinary in
[Nix](https://nix.dev), [Guix](https://guix.gnu.org) and
[Spack](https://spack.readthedocs.io). This page says which, and what each one
costs.

The honest summary: pkgx is excellent at *one build of each version*. Every
difficulty below is a consequence of that one assumption, and it is an
assumption an HPC site cannot make.

## 1. A package specification with no room for build options {#no-variants}

Upstream's own corpus — `docs.pkgx.sh/llms-full.txt`, read 2026-11-04 —
describes a grammar of **a name and a version constraint**, and nothing else:

```
node                  nodejs.org            github.com/gabime/spdlog
node^20.1.3           deno=1.35.3           "postgres>=12<14"
+openssl                                    # add to the environment
```

Against that page: *"The document describes no syntax for build options,
variants, or feature flags. It describes no syntax for compiler selection."*
In the C example, `+clang` is an ordinary `+pkg` addition — a package added to
the environment, not a compiler chosen for a build.

Compare Spack, whose
[spec syntax](https://spack.readthedocs.io/en/latest/spec_syntax.html) is built
around exactly this:

```
+debug                 # enable a boolean variant
~debug                 # disable it
compression=gzip       # single-valued
fabrics=verbs,ofi      # multi-valued: at LEAST these
fabrics:=verbs,ofi     # multi-valued: ONLY these
```

and Guix, whose
[package transformation options](https://guix.gnu.org/manual/html_node/Package-Transformation-Options.html)
do the same job from the other end — `--with-input` swaps a dependency through
the whole graph, `--with-c-toolchain` rebuilds a package and its dependents
with a different toolchain (the manual warns that this *"minimizes
incompatibility risks but cannot entirely eliminate them"*, naming the C++
standard library and OpenMP runtimes), and `--with-configure-flag` appends to
a package's configure flags.

This is not decoration. `+cuda`, `~mpi`, `fabrics=verbs,ofi` are the vocabulary
an HPC site uses to say what it wants, and without them there is no way to ask
for it.

go-zpkg adopts Spack's spelling rather than inventing one, with one
disambiguation forced by the filiation: **`~` means two things**. In pkgx it is
a version range (`~5.44` is `>=5.44 <5.45`); in Spack it disables a variant
(`~cuda`). A digit after the tilde is a range, a letter is a variant.

## 2. A store path that cannot hold two builds of one version {#one-build-per-version}

The same upstream page states the layout plainly:
`~/.pkgx/<fully-qualified-name>/v<version>/`, with `v*`, `v1` and `v1.2`
symlinks beside it. Deno 1.2.3 is at `~/.pkgx/deno.land/v1.2.3`.

A path keyed on the version **is a function of the version alone**, so two
builds of one version have one address. There is no bug to fix here; there is
simply nowhere to put the second one. It is the direct consequence of §1 —
with no way to *ask* for two builds, there was never a reason to *store* two.

Nix solves it with a digest in the path: a store path is a 20-byte digest in a
custom Base32 (*"20 arbitrary bytes become 32 ASCII characters"*) followed by a
hyphen and a readable name —
`/nix/store/b6gvzjyb2pg0kjfwrjmg1vfhh54ad73z-firefox-33.1`
([store path specification](https://nix.dev/manual/nix/2.28/store/store-path)).

Spack solves it with a DAG hash, computed from the JSON serialization of a
concrete spec's node dicts, and uses it *"both to compare specs and to create
unique installation directories for every possible configuration"*. Its manual
shows two `mvapich2@2.2%gcc` installations distinguished only by hash
(`qmt35td`, `er3die3`), and `/\<hash\>` substitutes for a spec in any command.

go-zpkg extends the pkgx layout rather than replacing it:

```
<store>/<project>/v3.3.11                      identity not recorded
<store>/<project>/v3.3.11-habcdefghij012345    THIS build of 3.3.11
```

The bare form is kept and it is not a default. It is the honest statement that
a bottle arrived without a spec annotation — mirrored, or built before the
factory published one — and that nothing on disk can say which build it is.
Minting a hash for it would put a claim in the path that no witness supports.

The hash is 16 characters of lowercase base32 because an OCI tag may contain
`[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}` and **not** `+`, so Nix's and Spack's
spellings of a spec cannot survive a round trip through a registry tag. The
same suffix is used in the tag and in the directory name, from one constant, so
the two cannot drift into two spellings of one idea.

## 3. An identity that is only a name {#identity}

Once a build is addressable, it needs to be *identifiable*: what makes this
bottle this bottle. go-zpkg hashes a **concrete spec** — platform, declared
variants, resolved closure, recipe plan digest, graft origin — in a
length-prefixed canonical form, so no two different specs can serialise to the
same bytes.

That is Spack's answer, arrived at the same way. Spack's hash
[was changed](https://github.com/spack/spack/pull/28504) to include build, run
and link dependencies *and* a canonical hash of the `package.py` file; the
older hash had excluded build dependencies. The lesson it encodes — that the
recipe is part of the identity, not merely its inputs — is one we took rather
than relearned.

pkgx has no such object, and could not use one: with no variants and no place
to put a second build, the identity of a package *is* its name and version.

## 4. A distribution that cannot be verified {#verification}

pkgx's `dist.pkgx.dev` serves tarballs over HTTPS. HTTPS authenticates the
*server*, not the *artifact*: it says the bytes came from that host today, not
that a particular build produced them.

go-zpkg publishes to an OCI registry with a Sigstore signature and an SLSA
provenance statement per bottle, and verification is on by default — a bottle
that cannot be verified is not installed, rather than installed with a warning.
Mirrored bottles are distinguishable from built ones by their provenance:
ours carries `externalParameters` and a versioned builder id, a mirror's
carries `{}` and an unversioned one.

Upstream remains reachable, as a **choice** rather than a default:
`ZPKG_DIST=https://dist.pkgx.dev` with `ZPKG_VERIFY=0`, which is the honest
pairing — unsigned distribution, verification off, said out loud.

## What this costs, and what we keep {#cost}

**Dropped:** four inherited decisions, each with the defect that made
dropping it cheaper than keeping it — [The break]({{< relref "the-break.md" >}})
goes through them one at a time. The largest:

The shared store. go-zpkg installs into `~/.zpkg`, not `~/.pkgx`.
Two reasons, and the second is the stronger: a hash in the path is not
something pkgx can read, so the trees could not stay interchangeable anyway —
and `~/.pkgx` is *pkgx's* directory, which upstream can be installed into on
the same machine. Sharing it was the collision the rename existed to remove,
left in the one place it does the most damage. Nothing is migrated: an
existing `~/.pkgx` is as much upstream's as ours, and `ZPKG_DIR=~/.pkgx` keeps
using it for anyone who wants that.

**Kept:** the pantry, the bottle format, the dependency grammar, the
relocatable prefix, and the ability to resynchronise with upstream when recipes
are added. Upstream's recipes are converted to HCL2 rather than forked, so a
periodic resync stays possible, and an improvement made here can be proposed
back.

**Not claimed:** go-zpkg does not have Nix's or Guix's functional purity. But
"not sandboxed" and "no time machine" are both vaguer than the code deserves,
so here is what is actually there and what is actually missing.

### The build *is* isolated, and here is how much {#isolation}

A factory build does not run on the host. `builder/sovereign-build.sh` enters
`unshare --mount --pid --fork` and then `chroot`s into a rootfs staged
entirely from our own bottles — that is what "FROM scratch build" means here:
neither the host's libc nor the host's compiler takes part, and the toolchain
is a reviewed, committed list (`builder/toolchain.txt`, now also a `members`
set in the pantry overlay) rather than whatever the distribution ships.

What that does **not** include, measured by searching for it rather than
assuming: no network namespace, no user namespace, no seccomp profile. A build
can reach the network. Nix's sandbox denies it outright and admits only
fixed-output derivations whose hash is declared in advance.

The gap is narrower than that sounds, because sources are pinned by content:
an archive by digest, a git recipe by commit (`bk`'s `source.go`,
`fetch/fetch.go`). What is missing is the *enforcement* — nothing stops a
build step from fetching something nobody declared.

### What a time machine would still need {#time-machine}

`bk lock` is already most of the way, and it was written against Guix's own
statement of the problem — *"to reproduce a profile bit-for-bit, manifests
alone might not be enough"*. It records the pantry revision, the overlay
revision, every resolved version, a per-project spec hash over
platform + version + parsed recipe + dependencies, and the version of `bk`
that produced it.

It records more than a Guix manifest has to, and for a reason: a pkgx recipe's
`versions:` asks GitHub for tags **at resolve time**, so pinning the pantry
revision is not enough. The same pantry commit resolved `tcl` to 9.0.4 and
then to 9.1.0 four hours apart. The resolved version has to be written down or
it is not pinned at all.

Four things stand between that and `guix time machine`:

1. **The lock pins inputs, not output bytes.** `bk lock`'s own documentation
   says so rather than implying otherwise: a bottle digest would pin them, and
   a lock taken before anything is built has no output to name.
2. **Nothing measures reproducibility.** No build runs twice and compares. Until
   it does, "the same inputs give the same bottle" is a hope with no instrument
   behind it — and `SOURCE_DATE_EPOCH` is not set anywhere.
3. **The lock does not name the builder image** it was built in, only the
   toolchain list that image is made from.
4. **Nothing re-executes the recorded `bk`.** The version is written down;
   fetching that version and running it is still a human step. `guix time
   machine` does exactly this and it is the part that makes the rest usable.

(1) and (3) are bookkeeping. (2) is the one that decides whether any of it is
true, and it is the next thing to build.

## Where the filiation is visible in the code {#visible}

| | |
| --- | --- |
| `bottle/pkgx.go` | pkgx's version grammar, constraints and bottle protocol |
| `bottle/concrete.go` | the concrete spec and its hash — Spack's idea |
| `bottle/storepath.go` | the store layout: pkgx's, with a hash where there is one |
| `PantryBase` | upstream's pantry, read directly |
| `ZPKG_DIST` | the distribution, defaulting to the signed registry |
