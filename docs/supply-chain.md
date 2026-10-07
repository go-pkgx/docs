# Supply chain

Every package in [`ghcr.io/go-pkgx/packages`](registry.md) ships with the evidence
needed to audit and trust it. Three artifacts are attached to each package as **OCI
referrers**, so they travel with the image and are discoverable from its digest:

- a **CycloneDX SBOM** — the package's components and versions
  ([`go-attest/sbom`](https://github.com/go-attest/sbom));
- an in-toto **SLSA provenance** statement — how and from what the package was built;
- a **cosign-style + minisign signature** over the package
  ([`go-attest/sign`](https://github.com/go-attest/sign)).

## What went IN, not only what came out

Those three describe the **output**. Until recently nothing recorded the input:
of 1858 recipes with a source URL (measured 2026-09; re-counted 2026-10-05 as
**1874 distributable URLs across 1899 recipes**, reaching **233 distinct
upstream hosts**), exactly one declared a checksum, and none was verified. A source tarball that changed upstream produced a different package,
correctly signed, attesting a URL rather than the bytes that came back from it.

The provenance now carries the source as a SLSA `resolvedDependency` — the
archive's SHA-256, or a git checkout's commit:

```json
"resolvedDependencies": [{
  "uri": "https://github.com/ROCm/ROCR-Runtime/archive/refs/tags/rocm-7.2.4.tar.gz",
  "digest": {"sha256": "60532cd86edce5a603aa18df406ec1f5a3d18f0663d79b3b7822ff721c5a04ec"}
}]
```

It records the candidate URL that **answered**, which for a recipe listing
mirrors is not always the first one — a build that fell through to a mirror and
one that did not are different builds.

And where a recipe declares a `sha:` URL — the form the pkgx format already had
and nothing read — the digest is verified. A mismatch fails the build rather
than falling through to the next mirror: a build that quietly succeeded from a
second host after the first served unexpected bytes would hide the one event
that check exists to surface.

## And where a recipe declares nothing, which is 903 of 904

Verifying a declared `sha:` is worth having and reaches almost nothing. Measured
on 2026-10-05 against a fresh `pkgxdev/pantry`: **one recipe of 904** carries a
`sha:` at all, and that one — `openssl.org` — points at `${{url}}.sha256`, a file
served by the **same host** as the tarball. It catches corruption in transit,
which TLS already catches. It does not catch substitution: a host serving
different bytes serves a matching `.sha256` beside them.

So the source mirror grew a second job. It already kept every archive a build
downloads, addressed by its SHA-256; it now also records **which digest each URL
served**, and a later fetch that disagrees is refused:

```
fetch: this URL has served different bytes before:
  https://…/x-1.2.tar.gz now serves 2c26b4…, and served 9f86d0… before
```

Trust on first use. It does not make an upstream trustworthy — it makes a
**change** visible, which is the part nobody had.

A **version bump is a first use**, because the version is in the URL: a new
release asks a question never asked before and is recorded rather than refused.
That is why there is no `--no-pin` escape; the case it would serve does not
arise.

The uncomfortable case is a mirror that cannot be **asked**. That is not an
absent pin, and treating the two alike is how trust-on-first-use quietly becomes
trust-every-time. They are distinguished, said out loud, and
`--source-pin-strict` chooses which is fatal — warn and continue by default,
because an unreachable registry is not evidence of tampering.

It is **opt-in**: `--source-mirror` is empty unless a run asks for it. Turning it
on is safe rather than risky — the mirror's existing contents carry only content
tags, so the first pass records and only the second enforces.

**Still missing**, and it is a decision rather than a patch: `openssl.org` also
declares `sig: ${{url}}.asc`, and nothing reads that field. It is the only
control in the recipe format a compromised host could not forge — the signing
key is not on `www.openssl.org` — and honouring it needs a key policy: whose
key, pinned where, and what happens when a release is signed by a new one. A
verifier that accepts any key proves only that somebody signed something.

## The pinned key

Signatures verify against a single pinned public key:

```
RWQ+rmH+fXy2iYr+gReQAOQtYWtH0A7UlxcAa2hpr+txNBwGqtpFsR6L
```

The factory signs with the matching private key (held only as a CI secret); every
consumer checks against this exact public key.

## Verify on install

The [`go-pkgx/bottle`](https://github.com/go-pkgx/bottle) backend that both `pkgx`
and `pkgm` import performs verification at install time:

- `bottle.VerifySignature` validates a package's signature against the pinned key.
- Setting **`PKGX_VERIFY=1`** makes verification **fail-closed**: a package with no
  signature, or one that does not verify, is **refused** rather than installed.

```sh
# refuses to install anything that isn't validly signed by the pinned key
PKGX_DIST=oci://ghcr.io/go-pkgx/packages PKGX_VERIFY=1 pkgm install lz4.org
```

This closes the loop end to end: the factory signs and attests at publish time, and
the consumer refuses to run anything it cannot verify.

## Metadata is not a program, and a key is not a caption

Signatures cover the **bytes**. The strings beside them — a project's name, its
summary, the commands it provides, its versions and platforms — reach your
terminal before any of that is checked, and they are written by people who are
not you: an upstream recipe author, whoever wrote the lock you were handed,
whatever registry `PKGX_DIST` points at.

Measured, with a crafted lock:

```console
$ pkgx --lock esc.lock.hcl true | cat -v
pkgx: no version of zlib.net satisfies "=1.3.2\x1b[2K\r…" (available: 2);
  asked for by =1.3.2^[[2K^Mzlib.net  1.9.9  ✓ verified (requested)
```

`ESC[2K` erases the line and `\r` returns the cursor, so on a terminal that line
rubs itself out and reprints as something reassuring. The same hole was in the
catalogue, where `pkgx ls` printed versions straight through.

Three treatments, and which one applies depends on **what the field is** and
**who writes it**:

| field | treatment | why |
| --- | --- | --- |
| project name | the whole file is **refused** | a name becomes a URL and a path; a bad one means somebody wanted something |
| summary, provides | **cleaned and shown** | cosmetic — refusing a catalogue over one paragraph would deny service for a typo |
| version, platform **in a lock** | the whole file is **refused** | exactness is the entire point of a lock |
| version, platform **in a catalogue** | that entry is **dropped**, the file is kept, and the count is printed | these come from an upstream recipe's `versions:`, so refusing would hand any recipe author a switch that turns off `pkgx ls`, `search` and every `<TAB>` for everyone |
| a tag from the registry | skipped | a registry is a configured endpoint |

A **key** can never be cleaned the way a caption is: a cleaned key would select
something other than what it says. So it is refused or dropped, never silently
repaired.

```console
$ pkgx catalog
~/.pkgx/catalog/darwin-aarch64.json
1908 project(s), 956 with dependencies, 2 hour(s) old
```

If a catalogue ever carried an unreadable version, a third line would say so —
`1 version string(s) were unreadable and are not listed`. On a catalogue our own
factory published it should never appear.

The allowlist was measured before it was written: across the two catalogues
published on 2026-10-06, **2801 version strings use fifteen distinct runes** and
the longest is 19 bytes. What is accepted is deliberately wider than that, because
a guard that refuses the ordinary case is a worse bug than the one it fixes.
