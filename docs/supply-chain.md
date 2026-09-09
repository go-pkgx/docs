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
of 1858 recipes with a source URL, exactly one declared a checksum, and none was
verified. A source tarball that changed upstream produced a different package,
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
