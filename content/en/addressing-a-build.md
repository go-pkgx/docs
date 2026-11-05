---
title: "Addressing a build"
weight: 18
---

# Addressing a build {#addressing-a-build}

A version is not an address. `fftw.org@3.3.11` built with CUDA and the same
version built without it are two different things that run differently, and
anything that calls them both `3.3.11` has to pick one and hope.

go-zpkg gives every build an **identity** — a hash of what it is — and puts
that identity in both places a build is named: its directory in the store, and
its tag in the registry.

```
store     <store>/fftw.org/v3.3.11-h7k2mq4xvbnd3rf5c
registry  ghcr.io/…/fftw.org:3.3.11-h7k2mq4xvbnd3rf5c
```

{{< callout type="info" >}}
**What ships today.** The identity, the hashed store prefix, the hashed
registry tag and the spec annotation are in `zbk` v0.63.0 and `bottle`
v0.81.0, and `zbk factory` publishes with them. `zpkgs builds` is in `zpkgx`
v0.36.0 and runs now.

**What does not.** Naming a build with `@version/<hash>` is **refused** by the
resolver rather than honoured: carrying the hash down to the store is
[zpkgx#142](https://github.com/go-zpkg/zpkgx/issues/142). Running *some* build
of a version under a name that says you chose one would be worse than saying
no, so it says no and points at `zpkgs builds`.

And until the registry is rebuilt with hashed tags, every published bottle
answers `nothing recorded` — which is the truth about the registry as it
stands, not a defect in the command.
{{< /callout >}}

## What the hash is of {#what-the-hash-is-of}

The platform, the resolved closure — every dependency at the exact version
**and build** it was made against — and a digest of the recipe.

Three consequences follow, and they are the reason it is built this way.

**It is a function of the inputs, never of the output.** The prefix is baked
into every binary the build produces, so it has to be known before a compiler
runs. Hashing the result into the result is circular. Nix hashes the
derivation and Spack the concrete DAG for the same reason.

**It is Merkle, for free.** A dependency is identified by the prefix it
resolved to, and that prefix carries the dependency's own hash — so hashing
the edge carries the whole dependency with it, transitively. It is the
mechanism Nix uses when a derivation references its inputs by store path.

**The recipe reaches it as a digest of the parsed value, not of the file.**
Reformatting a recipe, reordering its keys or rewriting its comments does not
move the hash; changing what it says does. Hashing the file's bytes instead
would make a reflowed comment rebuild the world — every dependent's hash
moves, every bottle is republished, and no binary differs. Spack strips
comments and docstrings from `package.py` before hashing for the same reason.

## The tag carries the address, the manifest carries the meaning {#tag-and-annotation}

A hash is opaque by construction. Nix pays for that: a store path says *which*
build and never *what* build, so answering "the one with CUDA?" needs a second
index.

So a published bottle carries its spec on its manifest:

```
io.github.go-zpkg.spec = {"v":1,"platform":"linux/x86-64","deps":[…],"recipe":"sha256:…"}
```

and a reader **recomputes** the hash from it and checks it against the tag. An
identity you cannot recompute is a claim, not a description; both halves come
from whoever could push to the registry, so reading the second without
checking it against the first would let a bottle tagged as the plain build
describe itself as the CUDA one.

Spack keeps the spec queryable the same way (`spack find libdwarf +debug`),
and conda writes its hash inputs to `info/hash_input.json` beside the hash.
Readability is a tooling property here, not a naming one.

A bottle that was only **mirrored** carries neither: no hash in its tag and no
annotation. We do not know how it was built, and a tag claiming an address
would say we do.

## Listing the builds of a version {#listing}

When more than one build of a version exists, the resolver refuses rather than
choosing:

```
fftw.org 3.3.11 was published as 2 different builds and nothing here says
which: a variant hash is not a version, so there is no newest one to pick
```

That refusal is only half a design without a way to answer it:

```console
$ zpkgs builds fftw.org@3.3.11
HASH              INSTALLED  PUBLISHED  SIGNED BY        WHAT
27o3ysxva6swayci  -          yes        go-zpkg          ⚠ withdrawn: CVE-2026-1234 → rnp44i · go.dev 1.27.3/aaaaaa
rnp44isdgdtzxcit  -          yes        go-zpkg, alice   go.dev 1.27.4/bbbbbb
```

One list joining the store and the registry, because a list of only what is
installed cannot answer *what else could I have*, and a list of only what is
published cannot answer *what do I already have*.

### The SIGNED BY column {#signed-by-column}

Two builders who build the same inputs get the **same** hash — deliberately —
so what tells them apart is who put their key on the result. The column names
them *as your own trust set names them*, never as the registry does: a name on
the artefact is written by whoever pushed it.

A `⚠` marks a row that does not meet your `ZPKG_TRUST_THRESHOLD` and will be
refused at install time. A `?` means the registry could not be asked, which is
**not** the same as *nobody signed it* — rendering the two alike is how a
network failure becomes a claim about the artefact. See
[Supply chain]({{< relref "supply-chain.md" >}}#two-builders-one-bottle).

### The WHAT column is the whole point {#what-column}

A column of bare hashes is unusable. A hash is opaque by construction, so a
list of two of them hands the reader back the problem the command was asked to
solve.

So each row says what **differs** — not what the build is. A full spec is
fifty-odd dependencies identical in all but one, and printing it per row
buries the one that matters. The axes are computed across all the builds
first, so two rows cannot describe themselves on different axes and read as
two unrelated facts rather than a comparison.

When nothing else differs, the **recipe** is named: a rebuild after a pantry
edit is one of the commonest reasons two builds of a version exist, and it is
invisible in the closure. With nothing printed those rows would be identical
and the list would look broken.

A build nothing records says `nothing recorded` rather than showing a blank
cell, which reads as a rendering fault.

A registry that could not be reached is an **error**, not an empty list — an
unobtained answer must not read as "nothing is published", because that is how
you conclude a build does not exist and rebuild it. What is installed is still
listed alongside the error: it is a true answer to half the question, and it
needs no network.

## Ordering them {#ordering}

`--sort=status|closure|hash|installed`.

`closure` is the one a reader means by "newer": a build made against a later
toolchain first. It is the comparison a CVE rebuild creates — a Go patch
release fixes a standard-library defect, nothing about the recipe or the
upstream version changes, only `go.dev` in the closure.

It is a **partial** order. One build is later than another only when it is at
least as new on every dependency they share and strictly newer on one. Two
builds that each lead on a different dependency are genuinely incomparable,
and declaring a winner would invent an order; they tie, and the tie falls back
to the hash. A build that gained a dependency the other does not have is a
different *shape*, not a later version of it — the CUDA build does not
supersede the plain one.

{{< callout type="warning" >}}
**There is no `date` key, and that is a property rather than a gap.** A
published bottle's timestamp is pinned — `zbk factory` stamps
`SOURCE_DATE_EPOCH`, or its own fixed epoch, never the wall clock — so that
re-publishing the same bottle yields byte-identical attestations and the same
referrer digests, which is what makes a re-push idempotent instead of
accumulating duplicates.

Every bottle therefore carries the **same instant**. Sorting by it would order
a list by a constant while looking like it ordered by time, and taking the
wall clock instead would make every rebuild a new artefact.

`--sort=installed` orders by when *this machine* got them, which is a fact
about your disk and not about the builds.
{{< /callout >}}

## Withdrawing a build {#withdrawing}

When a CVE catches a build, `ZPKG_WITHDRAWN` names a JSON file saying so:

```json
{"fftw.org": [{"version": "3.3.11",
               "hash": "27o3ysxva6swayci",
               "because": "CVE-2026-1234",
               "superseded_by": "rnp44isdgdtzxcit"}]}
```

**A build, not a version** — and that is the point. When the flaw is in the
*toolchain* a package was compiled with, the package is fine, the source is
fine, and the fix is another build of the **same version**. That is only
nameable because a build has an identity; it is the first thing this layout
buys that a version-addressed store could not do at all.

**It ranks, it does not filter.** A withdrawn build stays resolvable:

- two builds, one withdrawn → the other, and the withdrawal is announced;
- one build, withdrawn → that one, loudly, because there is nothing else;
- several left after the withdrawn ones are set aside → still ambiguous.

The reasoning is the same as for end-of-life release lines: a lock pins a
build, and a lock that stops resolving is a worse defect than the one being
fixed. Reproducibility and an explicit request outrank the preference.

An entry with **no hash** withdraws every build of the version — a flaw in the
source rather than in how it was compiled — and an entry naming one build wins
over it, so the build that *fixes* the flaw is not reported as carrying it.

The list is read locally and never from the network: the resolver must keep
working `FROM scratch` and air-gapped. Where the file comes from — `zbk
advisories` and the VEX documents attached to published bottles — is a
separate matter.

## Naming one {#naming-one}

A build is named by appending any prefix of its hash to the version, after a
slash — `fftw.org@3.3.11/7k2mq4`.

`/<hash>` is Spack's spelling, borrowed on purpose. Its manual: *"In place of
a spec in any command, substitute /`<hash>` where `<hash>` is any amount from
the beginning of a spec hash."* It is the one form that works without knowing
a package's variant vocabulary — which matters most exactly when a refusal has
just handed you two hashes and you have to pick.

Any prefix of the hash will do, as long as it names exactly one build. If it
matches two, that is a question rather than a choice, and you are told so with
both candidates listed so the next attempt can be longer.

{{< callout type="warning" >}}
**The `@version` is required before the hash.** A project name contains
slashes — a third of the pantry is named that way — so `x.org/pciaccess` has
to stay a project name. The syntax decides, not the shape of the trailing
component: without an `@version` in front of it, a trailing component is part
of the project name, always.
{{< /callout >}}

## Defaults live outside the store {#views}

A default version is **not** a property of the store. The store holds builds,
each at its own hashed path, and nothing in it says which one you prefer.

A default is a **view**: a directory of symlinks pointing into the store,
written where the chooser lives rather than where the builds live.

That separation is what makes the following work at once:

- **A shared store on NFS** can be mounted read-only and still serve a site.
  Each user's default is in their own profile, and choosing one writes nothing
  into the shared tree.
- **A container or microVM** built from an environment computes its links at
  creation time from the shared store, so the image gets exactly the builds
  that environment resolved — not whatever the host happened to default to.
- **Two environments on one machine** can prefer different builds of one
  version without either of them winning.

Nix makes the same separation with profiles, and for the same reason: *"One
cannot copy a store object to a store with a different store directory"* — a
cache stops being shareable the moment identity depends on where the reader
keeps it. The install prefix is therefore **not** part of the hash.

## Why there is no unhashed path {#no-unhashed-path}

There is no `<store>/<project>/v<version>` beside the hashed directories, and
no `v1` / `v1.2` / `v*` alias either — not even as a symlink, because a
symlink is a path too.

[The break]({{< relref "the-break.md" >}}) tells that story in full: the
aliases were inherited from pkgx, they only ever advanced, and installing an
**older** line of a project silently demoted `v2` onto it, so every consumer
bound to `v2` loaded the older library. The fix was careful, and it was a
careful fix to a problem created by having the alias at all.
