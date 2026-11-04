---
title: "The break: what we dropped from pkgx, and what it fixed"
weight: 16
---

# The break {#the-break}

[Filiation]({{< relref "filiation.md" >}}) says what go-zpkg inherits from
[pkgx](https://pkgx.sh) and what pkgx cannot do by design. This page is the
other half: the inherited things we **removed**, each with the defect that
made removing it the cheaper option.

Nothing here is a complaint about pkgx. Every item is a reasonable decision
for a tool that builds one version of each package once, and a wrong one for
a tool that must hold two builds of one version and say which is which.

## 1. A store path keyed on the version alone {#store-path}

**Inherited:** `~/.pkgx/<project>/v<version>/`, with `v1`, `v1.2` and `v*`
aliases beside it. Upstream's own documentation: *"All packages are
encapsulated in individual, versioned folders in `~/.pkgx`."*

**The defect.** A path keyed on the version is a function of the version
alone, so two builds of one version have one address. Installing the second
met the first and the installer refused:

```
already holds other content … the bottle carries v3.3.11
```

That refusal is correct behaviour on a wrong layout. `+cuda` and `~cuda` are
not a sequence, there is no newer one, and nothing in the path said which was
there.

It was not hypothetical. A rebuild is published as `3.3.11_1` and still
unpacks to `v3.3.11`, so installing it into a store holding the original
renamed onto an occupied directory, reported success, and installed nothing —
the caller then ran a binary under a prefix that had never been created and
got a bare `no such file or directory`.

**Now:** every installed tree is `v<version>-h<hash>`. The hash is the
build's identity — platform, resolved version, the canonical form of the
recipe, and the resolved prefix of every dependency. Two builds of one
version are two directories, and each says which it is.

**What survives, and why.** `v<version>` remains **as a symlink**, because it
is in the published binaries. Read out of a real bottle:

```console
$ otool -l …/zlib.net/v1.3.2/lib/libz.1.3.2.dylib
  name @rpath/zlib.net/v1.3.2/lib/libz.1.dylib
```

Every binary that links zlib names that path. A store without it breaks all
of them at load time, far from the cause. The hash goes on the real
directory; the published path keeps resolving.

That seam closes when the registry is rebuilt: a bottle built with a hashed
prefix carries the hash in its own `@rpath` references, and needs no alias at
all.

## 2. Somebody else's store directory {#store-directory}

**Inherited:** `~/.pkgx`, and `~/.pkgx/config.hcl2` beside it.

**The defect.** `~/.pkgx` is *pkgx's* store. Once our runtime is `zpkgx`,
upstream's `pkgx` can be installed on the same machine and the same PATH —
which is the entire reason for the rename. Two implementations writing one
directory, under two layouts that are no longer even the same, is the
collision the rename existed to remove, left in the one place it does the
most damage.

**Now:** `~/.zpkg`, `~/.zpkg/config.hcl2`, and the store's own marker file is
`.zpkg-platform`. **Nothing is migrated, on purpose**: an existing `~/.pkgx`
is as much upstream's as ours, and nothing in it says which bottles are
whose. `ZPKG_DIR=~/.pkgx` keeps using the old one for anyone who wants that.

## 3. A vendor namespace that is not ours {#namespaces}

**Inherited:** `application/vnd.pkgx.bottle.layer.v1.tar+zstd`,
`application/vnd.pkgx.source.layer.v1`, `dev.pkgx.source.uri`.

**The defect.** Checked against upstream's own corpus rather than assumed:
pkgx distributes **plain HTTPS tarballs** from
`dist.pkgx.dev/<pkg>/<os>/<arch>/` with `.sha256sum` and `.asc` beside them.
It defines **no OCI registry, no artifact type and no media type at all**.

The OCI packaging is entirely ours — our factory invented it — and we had
minted its names inside a vendor space whose owner never agreed to them. A
name that looks like yours because you are the only one using it is not
yours.

Our own keys had the same flaw one level down: the annotations were
`org.go-pkgx.*`, and the rename turned them into `org.go-zpkg.*`. A
reverse-DNS key is a claim on a **domain**, and nobody here has registered
`go-pkgx.org` or `go-zpkg.org`. The rename carried an unbacked claim across
rather than fixing it.

**Now:** `application/vnd.go-zpkg.*` for media types and
`io.github.go-zpkg.*` for annotations — the reverse-DNS of a host that
demonstrably answers, which is the convention OCI itself uses
(`org.opencontainers.image.*`) applied honestly.

**Read compatibility is asymmetric, and deliberately so.** Layers are matched
by their codec suffix (`tar+zstd`), never by the whole string, so both
spellings read without the reader being told about either. Annotations are
matched by exact key, so the older spellings are still read — including the
intermediate `org.go-zpkg.*`, which shipped in five releases. The failure
mode is why: a missed annotation key does not raise an error, it quietly
concludes the bottle has no glibc flavour, no declared sonames and no
recorded spec.

## 4. A distribution that cannot be verified {#verification}

**Inherited:** `https://dist.pkgx.dev`, tarballs over HTTPS.

**The defect.** HTTPS authenticates the *server*, not the *artifact*: it says
the bytes came from that host today, not that a particular build produced
them.

**Now:** an OCI registry with a Sigstore signature and an SLSA provenance
statement per bottle, and verification **on by default** — a bottle that
cannot be verified is not installed, rather than installed with a warning.
Upstream stays reachable as a *choice*: `ZPKG_DIST=https://dist.pkgx.dev`
with `ZPKG_VERIFY=0`, which is the honest pairing said out loud.

## What this costs {#cost}

Every bottle published before the break has to be **rebuilt**, not copied. A
copy would carry the old internal paths, the old media types and no identity
— it would move the defects rather than end them.

That is the whole reason the break happens now rather than later: the
registry is the largest thing that would have to change, and it is cheaper to
rebuild 1 967 projects once than to carry four inherited defects indefinitely
and explain them to everyone who meets one.
