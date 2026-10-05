# Finding packages

A package manager you cannot browse is a package manager you have to already
know. `pkgx ls` walks the tree; `<TAB>` completes into it.

```console
$ pkgx ls
curl.se                                  8.17.0, 3 under
github.com                               452 under
gnu.org                                  54 under
zlib.net                                 1.3.2

$ pkgx ls gnu.org
gnu.org/autoconf                         2.73
gnu.org/bash                             5.3
gnu.org/gcc                              16.2.0, 1 under
…

$ pkgx ls zlib.net
zlib.net is a package, not a namespace — 1.3.2 1.3.1
```

A node can be **both** a package and a namespace. `curl.se` is one — it is a
package, and `curl.se/ca-certs` lives under it — so the listing says both
rather than picking one.

## Completion

```sh
eval "$(pkgx completion bash)"     # or zsh, or: pkgx completion fish | source
```

```console
$ pkgx +gnu.org/ba<TAB>
+gnu.org/bash

$ pkgx gnu.o<TAB>
gnu.org/          # with the slash, so the next TAB descends
```

The snippet is a few lines and does nothing but call `pkgx`. Spack generates
`spack-completion.bash` from its command tree and commits it; Guix
hand-writes one per shell; both then need a check that the file still matches
the program. Nix took the other road — the binary answers when
`NIX_GET_COMPLETIONS` names the argument being completed — and that is the
one followed here.

The reason is specific: what is being completed is not a fixed command tree,
it is **the registry**, which changes without pkgx changing and which a
generated file could never be level with. It is also the only shape that
works in a `FROM scratch` image, where there is no completion framework, no
python and no generator to run.

## Where the list comes from

`gnu.org/<TAB>` has to know what exists, and **a registry cannot be asked**:

```
GET /v2/_catalog                             403   (ghcr issues no token)
GET /v2/go-pkgx/packages/zlib.net/tags/list  200   ["1.3.2", …]
```

Versions, yes. Projects, no.

Nix, Guix and Spack all answer this from a **local index** — an eval cache, a
channel checkout, a package repo. The pkgx-native equivalent is an artefact
in the registry itself, so the registry carries a **catalogue**:

```console
$ bk catalog --pantry … --overlay … -o catalog.json
catalog: 1908 project(s) → catalog.json
```

1908 projects, 905 roots, **100 KB**, built in 0.4 s with no network. It is
an **ordinary bottle** — signed, attested, mirrored and cached like any
other, fetched in one pull — and it is published **per platform**, because
what is available differs by architecture and a single list would tell most
readers that packages are available which, for them, are not.

## When it cannot be read

`pkgx ls` falls back to **what is installed**, and says so in its header:

```console
$ pkgx ls
pkgx: showing what is INSTALLED — the registry catalogue could not be read
```

Offline, and in a fresh scratch image before the first fetch, that is not a
lesser answer: it is the only true one available, and it is often the one
wanted, because completing onto something already installed costs nothing to
run.

The two lists are **never merged**. "Available" about a mix of a registry and
a local store is a word with no meaning, and a header cannot be honest about
a list with two sources.
