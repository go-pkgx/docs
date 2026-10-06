# Finding packages

A package manager you cannot browse is a package manager you have to already
know. `pkgx ls` walks the tree; `<TAB>` completes into it.

```console
$ pkgx catalog update
1908 project(s) → /home/you/.pkgx/catalog/linux-x86-64.json

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

**It needs no network** — not once `pkgx catalog update` has run, and never
again until you run it. `pkgx --graph` answers the same question by
resolving against the registry, which is the better answer when you have one
and no answer at all in a `FROM scratch` image that has not fetched anything
yet.

What it shows is what recipes **declare**. It is not the installed closure:
`bottle` also pulls providers by soname that no recipe names, so a real
install can hold more than this tree does.

## Everything is named; what is here is marked

```console
$ pkgx ls doxygen.nl
doxygen.nl — no bottle here

$ pkgx ls --tree curl.se
curl.se — 8.17.0
  curl.se/ca-certs  2026.09.25
  doxygen.nl  (no bottle here)
  openssl.org  3.6.0
```

A catalogue is published **per platform**, because what is available
differs by architecture — the s390x lane has a fraction of what
linux/x86-64 has.

**Dropping the name** would tell you the project does not exist, which is
false. **Printing it bare** tells you nothing. **Marking it** tells you it
exists, that there is no bottle for you, and therefore that building it is
the thing to do next.

In a tree, that mark is usually the most useful line on the page: an
unsatisfiable dependency is the reason the thing above it cannot be
installed, and it would otherwise print as a bare name among satisfied
ones. It reaches `<TAB>` too, as the description beside the candidate —
the moment it is worth knowing, *before* the name is typed rather than
after it fails.

What is behind the mark: the factory asks the registry, per project, which
versions it carries and whether this platform is among them. Two things it
deliberately does **not** do. It does not borrow: a version list that fell
back to the upstream `dist.pkgx.dev` is dropped, because "available" is a
claim about **one** registry and a list from another one answers a
different question. And it does not assume: a tag listing spans every
architecture, so a mirror wave that lands x86-64 publishes a tag s390x
cannot use, and the version recorded is the newest this platform really
carries.

## What you already have

```console
$ pkgx ls --tree curl.se
curl.se — 8.17.0  ✓
  curl.se/ca-certs  2026.09.25  ✓
  doxygen.nl  (no bottle here)
  openssl.org  3.6.0  ✓ 3.5.1
```

A catalogue says what **exists**; a reader standing in front of it mostly
wants to know what they **have**. nix and guix show the state of the store
in everything they print.

A bare `✓` when what is installed is the version on offer, and `✓ <version>`
when it is **not** — the case worth the extra word, because it is the one
where a command would fetch something new.

## You know the command, not the package

```console
$ pkgx search rg
crates.io/ripgrep                        rg  14.1.1
aardvark.io/rgbds                        rgbasm  0.9.0
```

`rg` is `crates.io/ripgrep`. No guess at the project name reaches it, and
`<TAB>` — a **prefix** on the project path — never will either. Completion
and search are different tools, and that is the line between them.

`nix search`, `guix search` and `spack list -s` all match a package's
**description**, which is right for a collection that has descriptions. This
pantry does not. Counted on the catalogue published for linux/aarch64 on
2026-10-06: of its **1908** projects, **1582** name the commands they put on
PATH and **six** carry a summary. A search over prose would find six
packages.

So the match is on names and on commands, ranked by *how* a thing matched
rather than by how often — a command named exactly what you typed is what
you meant, every time. Nothing found exits **non-zero**, so
`pkgx search x || echo none` works.

## Walking it full-screen

```console
$ pkgx browse
gnu.org — 54

> gnu.org/bash                             5.3  ✓
  gnu.org/gcc                              no bottle here
 +gnu.org/make                             4.4.1

↑ ↓ move · → descend · ← back · tab names ⇄ dependencies · / search
· space pin · q quit, printing what is pinned
```

[`nix-tree`](https://github.com/utdemir/nix-tree) is the precedent: the same
data read one node at a time is data nobody explores. Descending into a
**leaf** shows what it needs, rather than making you press `tab`.

`q` prints what you pinned on **stdout** while the screen goes to stderr, so
`pkgx +$(pkgx browse)` composes. It reads the cached catalogue and nothing
else — 0.013 s against the real 1908-project one — and **without a terminal
it prints the tree and exits**, because `pkgx browse | less`, a CI log and a
scratch image are the same case.

## Running exactly what a lock pins

```console
$ pkgx --lock seed.lock.hcl -- make
pkgx: seed.lock.hcl, 231 pin(s), 3 hour(s) old
```

A lock nobody consumes is a record, not a mechanism: cargo's `--locked` and
npm's `ci` exist because the file only means something once a command
refuses to deviate from it.

The defect is not hypothetical here. The **same** pantry commit resolved tcl
to 9.0.4 and then to 9.1.0 four hours apart, because a recipe's `versions:`
asks GitHub at resolution time.

**Every pin becomes a root**, not just the lock's own — that is what makes
it a lock rather than a hint, and a set that cannot be satisfied exactly
fails instead of materialising something near it. A lock taken on another
platform is refused by name: **537 of 1908** projects available on
`linux/x86-64` have no bottle for `darwin/aarch64`.

An environment is still **not** a lockfile. An environment names a *set* and
resolves it afresh; a lock names *versions*. Two paths, side by side.

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

Six lanes publish one each: `linux/x86-64`, `linux/aarch64`, `linux/s390x`,
`darwin/aarch64`, `darwin/x86-64` and `windows/x86-64`. A platform with no
catalogue would give you silence, which is not the same answer as "nothing
is available here".

Signed, and **checked**: forging a catalogue does not get an unsigned bottle
installed, because the install path verifies anyway. It gets a near-miss
name offered at your prompt, which is the whole of a typosquat. The attack
is on the reader.

## One command fetches it

`pkgx catalog update` is the **only** thing that asks the registry what
exists. `ls` and `<TAB>` read the file it wrote, and nothing on that path
opens a socket.

That line is the difference between a completion you leave switched on and
one you turn off. A `<TAB>` is a fresh process — there is no state between
two presses — so a fetch on the read path is paid again on every press, in
full. Measured, one press of `<TAB>` on `gnu.o`:

| | median | answer |
| --- | --- | --- |
| fetching on every press | 295 ms | nothing |
| reading the file | **7.6 ms** | `gnu.org/` |

`guix pull` and `nix-channel --update` put the line in the same place.

The price is that the index goes stale and nothing tells you by magic, so
every command that reads it says how old it is:

```console
$ pkgx catalog
/home/you/.pkgx/catalog/linux-x86-64.json
1908 project(s), 986 with dependencies, 2 hour(s) old
```

## Shipping a catalogue with the image

`PKGX_CATALOG=<file>` reads one from somewhere else entirely — for an
air-gapped image, or to inspect a catalogue before publishing it. An image
that ships one can also simply drop the file at the path `pkgx catalog`
prints, and never fetch at all.

Set and unreadable is a **refusal**, not a quiet fall back to the local
store: somebody who names a file means that file, and answering from
somewhere else under a header blaming the registry would be three wrong
things in one line.

## When there is none

`pkgx ls` falls back to **what is installed**, and says so in its header —
with the sentence that fits the case it is actually in:

```console
$ pkgx ls                      # never fetched one
pkgx: showing what is INSTALLED — no catalogue yet; run: pkgx catalog update

$ pkgx ls                      # one is there and will not parse
pkgx: showing what is INSTALLED — /home/you/.pkgx/catalog/linux-x86-64.json could not be read
```

Those two are not the same state, and "run `pkgx catalog update`" is the fix
for the first and no help at all for the second. Sending somebody to a
command that cannot work is worse than saying nothing.

Offline, and in a fresh scratch image before the first fetch, the fallback
is not a lesser answer: it is the only true one available, and it is often
the one wanted, because completing onto something already installed costs
nothing to run.

The two lists are **never merged**. "Available" about a mix of a registry and
a local store is a word with no meaning, and a header cannot be honest about
a list with two sources.
