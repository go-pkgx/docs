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

$ pkgx ls curl.se
curl.se — 8.17.0
  curl.se/ca-certs
  nghttp2.org
  openssl.org
  zlib.net

also a namespace, 3 under it: pkgx ls curl.se/
```

## A node has two kinds of thing under it

`gnu.org` has `gnu.org/bash` under it because of how it is **named**.
`curl.se` has `openssl.org` under it because of what it **needs**.

The same words — "what is available under this node" — mean *containment*
at a namespace and *dependency* at a package, and `ls` answers whichever
the node is. Nix, Guix and Spack keep the two in separate commands
(`guix graph`, `nix-tree`); here the node decides, because the person
typing already knows which kind of thing they named.

A node can be **both**, and `curl.se` is. A **trailing slash** asks for the
namespace — the only way to reach `curl.se/ca-certs` from `curl.se` — and a
node that is both says where its other half is rather than hiding it.

```console
$ pkgx ls --tree curl.se
curl.se — 8.17.0
  curl.se/ca-certs
  nghttp2.org
  openssl.org
    curl.se/ca-certs  (shown above)
  zlib.net
```

`--depth N` bounds the descent. `guix graph --max-depth` exists for the same
reason: the full transitive graph of anything interesting is pages long, and
the first level is what a person reads. A diamond is expanded once and named
once — and the one expanded is the **direct** dependency, because that is the
one a reader came for.

**It needs no network.** `pkgx --graph` answers the same question by
resolving against the registry, which is the better answer when you have one
and no answer at all in a `FROM scratch` image that has not fetched anything
yet.

What it shows is what recipes **declare**. It is not the installed closure:
`bottle` also pulls providers by soname that no recipe names, so a real
install can hold more than this tree does.

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

## Shipping a catalogue with the image

`PKGX_CATALOG=<file>` reads one from disk instead of the registry — for an
air-gapped image, or to inspect a catalogue before publishing it.

Set and unreadable is a **refusal**, not a quiet fall back to the local
store: somebody who names a file means that file, and answering from
somewhere else under a header blaming the registry would be three wrong
things in one line.

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
