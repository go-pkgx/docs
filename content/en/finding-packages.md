---
title: "Finding packages"
weight: 40
---

# Finding packages {#finding-packages}

A package manager you cannot browse is a package manager you have to already
know. `pkgs ls` walks the tree; `<TAB>` completes into it.

```console
$ pkgs catalog update
1908 project(s) → /home/you/.pkgx/catalog/linux-x86-64.json

$ pkgs ls
curl.se                                  8.17.0, 3 under
github.com                               452 under
gnu.org                                  54 under
zlib.net                                 1.3.2

$ pkgs ls gnu.org
gnu.org/autoconf                         2.73
gnu.org/bash                             5.3
gnu.org/gcc                              16.2.0, 1 under
…

$ pkgs ls curl.se
curl.se — 8.17.0
  curl.se/ca-certs
  nghttp2.org
  openssl.org
  zlib.net

also a namespace, 3 under it: pkgs ls curl.se/
```

## A node has two kinds of thing under it {#a-node-has-two-kinds-of-thing-under-it}

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
$ pkgs ls --tree curl.se
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

**It needs no network** — not once `pkgs catalog update` has run, and never
again until you run it. `pkgx --graph` answers the same question by
resolving against the registry, which is the better answer when you have one
and no answer at all in a `FROM scratch` image that has not fetched anything
yet.

What it shows is what recipes **declare**. It is not the installed closure:
`bottle` also pulls providers by soname that no recipe names, so a real
install can hold more than this tree does.

## Everything is named; what is here is marked {#everything-is-named-what-is-here-is-marked}

```console
$ pkgs ls doxygen.nl
doxygen.nl — no bottle here

$ pkgs ls --tree curl.se
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

## How many are marked, and how to ask {#how-many-are-marked-and-how-to-ask}

The mark is not rare. Measured against the published `darwin/aarch64`
catalogue on 2026-10-23, **567 of 1940 projects** have a recipe and no
bottle — 29% of the pantry. `pkgs ls` marks each one as you meet it; to see
them as a list rather than one at a time:

```console
$ go run ./internal/catstat -missing ~/.pkgx/catalog/darwin-aarch64.json
567 of 1940 project(s) have no bottle here
aerc-mail.org
agpt.co
…
```

A percentage is not a work list: nobody can file, build or triage 29%.

When you want one of them, `pkgs request` turns "this is missing" into a
report somebody can act on — and **sends nothing**:

```console
$ pkgs request aerc-mail.org
project   aerc-mail.org
platform  darwin/aarch64
pkgs      0.26.0
state     published for linux/aarch64, NOT for darwin/aarch64

Nothing was sent. `pkgs request --url aerc-mail.org` prints a link that opens this as an issue.
```

`--url` prints a prefilled issue link you read first. There is no flag that
posts and nothing to configure or forget: the only way a report reaches
GitHub is a person clicking a link they have already seen in full.

The payload is a **fixed list** — project, platform, tool version, state —
assembled by a function that does not import `os`, so it cannot reach argv
or the environment even by accident. That is deliberate. The command that
prompted all of this was

```console
$ pkgx openstack user list --domain stratuslab
```

and a reporter that attached "the command that failed" would have published
an organisation's infrastructure names into a public tracker, from a
machine whose owner was debugging rather than publishing.

A request for something **already here** is refused rather than filed: an
issue whose premise is false spends a maintainer's trust.

### The failure says where to go {#the-failure-says-where-to-go}

That only helps somebody who knows the command exists, so the failure
itself names it:

```console
$ pkgx +aerc-mail.org -- echo ok
pkgx: no version of aerc-mail.org is published for darwin/aarch64 (available: 0.22.0; not published here: 0.22.0)
pkgx: if it should be here: `pkgs request aerc-mail.org` prints a report you can open as an issue (it sends nothing)
```

The hint fires on a **type**, not on words in a message. An earlier version
matched the text and turned up on

```console
pkgx: command not found in the requested packages: true
```

where the package was published and the *command* was missing — and a hint
that appears on unrelated failures teaches people to ignore hints. A build
that failed, a signature that did not verify and a registry having a bad
day get none: none of them is something to ask a maintainer to publish.

### And it remembers {#and-it-remembers}

The hint fires once, in front of somebody who is in the middle of something
else. `pkgs request` answers about one project at the moment you ask it.
Neither can tell you that the same package has failed eleven times this
month, which is the thing worth acting on:

```console
$ pkgs journal
2 failure(s) recorded here, 1 of them a package this registry does not have

most asked for and not here:
     1  aerc-mail.org

`pkgs request <project>` turns one of those into a report you can open as
an issue. It sends nothing, and neither does this.
```

It is a **local file**, and that is the whole design. The useful signal from
a package manager is which packages people reach for and do not get, and the
usual way to collect it is to phone home. This does not: it keeps the promise
`request` already makes a few paragraphs above, and adds the only thing
`request` cannot — memory. `PKGX_JOURNAL=0` records nothing at all, and
`pkgs journal --clear` removes it.

A record of failures is still a thing that can go wrong, so three properties
are tested rather than intended:

| | |
|---|---|
| it cannot fail the command it records | a store that cannot be written is a reason for the journal to stay empty, not for `pkgx node` to fail |
| it cannot grow without bound | a failing command in a loop would otherwise fill the disk — one rotation, capped |
| it cannot carry what it cannot print | the text is an error string quoting names read out of a catalogue, a registry index or a downloaded binary, so it is sanitised on the way **in** |

A fourth came from the test rather than the design: a crash mid-write leaves
a line with no newline, and appending onto it glued the next entry to the
damaged one — one interrupted write costing two records.

## What you already have {#what-you-already-have}

```console
$ pkgs ls --tree curl.se
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

## And what that has cost you {#and-what-that-has-cost-you}

A `✓` is per project. `pkgs store` is the whole disk:

```console
$ pkgs store --by-size -n 6
~/.pkgx
llvm.org                                   13.8 GiB  3 versions, 8.5 GiB not the newest here
android.com/cmdline-tools                   6.4 GiB
swift.org                                   4.6 GiB
github.com/coqui-ai/TTS                     1.6 GiB
go.dev                                      1.5 GiB  7 versions, 1.3 GiB not the newest here
gohugo.io                                1020.5 MiB  13 versions, 934.6 MiB not the newest here
… 296 more

41.2 GiB over 363 version(s) of 302 project(s)
12.8 GiB is in versions that are not the newest present — which is not the same as unused
```

Every ephemeral run leaves its bottles behind and **nothing removes one**.
`pkgm list` answers a different question — what you asked to have installed —
so a bottle pulled six months ago to run one command is in neither. Those
figures are one developer's machine on 2026-10-08, and **thirteen versions of
gohugo.io** is not a number anybody chose.

### It reports and deletes nothing {#it-reports-and-deletes-nothing}

That is deliberate, and it is the same line every comparable tool draws:
[`nix path-info -S`][npi] against `nix store gc`, `guix gc --list-live`
against `guix gc`, `spack find` against `spack gc`. All three make the
**roots** explicit before removing anything.

[npi]: https://nix.dev/manual/nix/2.35/command-ref/new-cli/nix3-path-info

There is no profile here, so there is **no root set**: a version that looks
superseded may be exactly what a lock pins or what an environment asks for.
So the last line states a fact about the disk and says in as many words that
it is **not the same as unused**.

### Two things about the figures {#two-things-about-the-figures}

It **walks the tree**, so it costs seconds rather than milliseconds — eight
over 41 GiB. That is why it is its own command and not a line in `pkgx
catalog`: nothing on the `<TAB>` path may cost that.

Sizes are **powers of 1024** and say so with the `i`. `du` counts blocks and
this counts file sizes, so the two disagree twice over; naming the unit is
the least a report can do about that.

### And what any of it is for {#and-what-any-of-it-is-for}

A size is not a decision. `--root` says what a lock needs:

```console
$ pkgs store --root seed.lock.hcl
…
41.2 GiB over 363 version(s) of 302 project(s)

against 1 root(s): 12.3 MiB live over 7 version(s), 41.2 GiB in 356 version(s) no root needs
dead here means NOT REACHABLE FROM THE ROOTS YOU NAMED, and nothing is removed
```

[`guix gc`][gc] draws the line this copies: *"any file under /gnu/store
reachable from a root is considered live and cannot be deleted; any other file
is considered dead"*. What makes that safe is that the **roots are explicit** —
guix keeps them as symlinks under `/var/guix/gcroots`, and `--list-live`
reports before `gc` removes.

[gc]: https://guix.gnu.org/manual/html_node/Invoking-guix-gc.html

**A lock is already a root set.** `pkgx --lock` pins every project in the
closure, not only what you typed, so membership in it *is* reachability — no
graph walk and no network. An **environment** would not do: it names
constraints, which have to be resolved against the registry, and this command
works offline.

Three things worth knowing:

- the **version** counts, not just the project: llvm 16.0.6 is dead while
  22.1.8 is live;
- several `--root` flags **union**, because that is what *reachable from any of
  them* means;
- **without `--root` there is no live/dead line at all** — "nothing is reachable
  from nothing" is true and would read as a verdict on your disk.

A mistyped `--root` is refused **before** the store is walked: eight seconds
over 41 GiB is too long to wait to be told the arguments were wrong.

## The two questions that do not descend {#the-two-questions-that-do-not-descend}

Everything above descends. `ls --tree` says what a project **needs**, which is
what you ask before installing it. Two questions point the other way, and no
amount of descending reaches them.

**Who needs this** — what you ask before *changing* something:

```console
$ pkgs ls --dependents openssl.org
openssl.org — 4.0.2  ✓ 3.6.0
  agwa.name/git-crypt  (no bottle here)
  apache.org/arrow  25.0.1
  …

runtime dependents on darwin/aarch64; a build-only user is in no catalogue
```

The number is the point — it is the blast radius of a bump, and before this it
could not be obtained at all. Counted on the catalogues published 2026-10-06,
**and it is not one number**:

| platform | need openssl directly | transitively (`--tree`) |
| --- | ---: | ---: |
| darwin/aarch64 | 153 | 789 |
| linux/aarch64 | 164 | 854 |

Out of 1908 projects in each. The two differ because a catalogue records what a
platform really carries, so a project with no bottle here has no edges here. That
is why the command prints the platform under the list rather than leaving the
figure to be quoted bare.

**Why is this in here** — what you ask when a closure holds a surprise:

```console
$ pkgs why git-scm.org zlib.net
git-scm.org — 2.55.0  ✓ 2.47.1
  → zlib.net — 1.3.2  ✓
```

A **shortest** path, as [`nix why-depends`][why] answers it: the point is an
explanation a person can hold, not every route. When there is none it says so
and exits 1, which is the answer that tells you a bump cannot reach you.

[why]: https://nix.dev/manual/nix/2.35/command-ref/new-cli/nix3-why-depends

Every comparable tool has the first question and they do not all mean the same
thing by it. [`spack dependents`][sd] is direct, or transitive with `-t`.
[`guix refresh --list-dependent`][gr] is about what would need **rebuilding**,
and its manual says plainly that it *only approximates* that.

[sd]: https://spack.readthedocs.io/en/latest/
[gr]: https://guix.gnu.org/manual/html_node/Invoking-guix-refresh.html

### What this answer is not {#what-this-answer-is-not}

The catalogue carries **runtime** dependencies, reduced for **one platform**. So
`--dependents` is the runtime blast radius on the platform you are standing on:
who would load different bytes if this project changed.

**It is not the rebuild set.** A build dependency is paid once by the factory
and is in nobody's installed closure, so it is not in a catalogue and cannot be
in this answer. Guix is about rebuilds and says it approximates them; this is
about closures and is exact for what it covers. The line is printed under the
list, not only written here, because a number is read where it is printed.

And `why` answers from the **declared** graph. Nix answers from the store, by
scanning built output for the hash parts of store paths — there is nothing to
scan before anything is installed, and saying which it is beats implying a scan
nobody ran.

## You know the command, not the package {#you-know-the-command-not-the-package}

```console
$ pkgs search rg
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

Those figures are not a recollection: `go run ./internal/catstat
~/.pkgx/catalog/*.json` in [go-pkgx/pkgx] prints them from the file you
have, through the same parser the client uses, so you can check them
against your own catalogue rather than against this page. Both published
platforms agree on the number that matters — six — and differ where you
would expect them to, 1430 projects with a bottle on linux/aarch64 against
1371 on darwin/aarch64.

[go-pkgx/pkgx]: https://github.com/go-pkgx/pkgx

So the match is on names and on commands, ranked by *how* a thing matched
rather than by how often — a command named exactly what you typed is what
you meant, every time. Nothing found exits **non-zero**, so
`pkgs search x || echo none` works.

## Walking it full-screen {#walking-it-full-screen}

```console
$ pkgs browse
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
`pkgx +$(pkgs browse)` composes. It reads the cached catalogue and nothing
else — 0.013 s against the real 1908-project one — and **without a terminal
it prints the tree and exits**, because `pkgs browse | less`, a CI log and a
scratch image are the same case.

## Running exactly what a lock pins {#running-exactly-what-a-lock-pins}

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

## Completion {#completion}

```sh
eval "$(pkgs completion bash)"     # or zsh, or: pkgs completion fish | source
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

## Where the list comes from {#where-the-list-comes-from}

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

Seven lanes publish one each. **Do not read the list from here** — this
sentence has already been wrong once, when the seventh lane was added and
nothing in this file knew. Ask the registry, which is the only thing that
knows:

```console
$ pkgs catalog platforms
darwin/aarch64
darwin/x86-64
linux/aarch64
linux/s390x
linux/x86-64
wasip1/wasm
windows/x86-64
```

A platform with no catalogue would give you silence, which is not the same
answer as "nothing is available here". `wasip1/wasm` is the odd one and has
[its own page]({{< relref "wasm.md" >}}): its modules are statically linked, so
a catalogue entry there names build-time dependencies that are not themselves
published for that platform.

Signed, and **checked**: forging a catalogue does not get an unsigned bottle
installed, because the install path verifies anyway. It gets a near-miss
name offered at your prompt, which is the whole of a typosquat. The attack
is on the reader.

## One command fetches it {#one-command-fetches-it}

`pkgs catalog update` is the **only** thing on the read path that asks the
registry what exists — `catalog platforms` below asks it too, and is the one
other command that does. `ls`, `search`, `why`, `browse` and `<TAB>` read the
file `update` wrote, and nothing on that path opens a socket.

That line is the difference between a completion you leave switched on and
one you turn off. A `<TAB>` is a fresh process — there is no state between
two presses — so a fetch on the read path is paid again on every press, in
full. Measured, one press of `<TAB>` on `gnu.o`:

| | median | answer |
| --- | --- | --- |
| fetching on every press | 295 ms | nothing |
| reading the file | **7.6 ms** | `gnu.org/` |

`guix pull` and `nix-channel --update` put the line in the same place.

### A platform you are not on {#a-platform-you-are-not-on}

Both `catalog update` and `--graph` take `--platform os/arch`, because the
answer is **not the same everywhere** and the question is usually asked from
whichever machine you happen to be sitting at.

```console
$ pkgs catalog update --platform linux/x86-64
1915 project(s) → ~/.pkgx/catalog/linux-x86-64.json
```

It writes to that platform's own file, so `ls` and `<TAB>` — which read the
host's — cannot mistake a foreign catalogue for yours.

And which platforms are there at all:

```console
$ pkgs catalog platforms
darwin/aarch64
darwin/x86-64
linux/aarch64
linux/s390x
linux/x86-64
windows/x86-64
```

⛔ **This exists because a list copied into a file froze while the registry
grew.** `--platform` could be *guessed at* one value at a time — an
unpublished platform is refused with `platform not in index`, so the index
plainly knows — and nothing could ask it for the set. The daily vulnerability
scan carried its list in a workflow file, that list said two, and the registry
published six at the time (seven now, which is the point); see
[Supply chain]({{< relref "supply-chain" >}}).

The output is an **input**: one platform per line and nothing else, ready to
hand back to `--platform` from a loop. A failure is **not an empty list** — a
loop that reads nothing and sees success scans nothing and calls it clean — so
an unreadable index, a tag that is a single manifest, and an index naming no
platform are each an error with its reason on stderr.

`--graph` answers the sharper question, the one a recipe author has:

```console
$ pkgx --graph +go.dev@^1.27.1
go.dev 1.27.1
└─ openssl.org 3.6.4

$ pkgx --graph --platform linux/x86-64 +go.dev@^1.27.1
pkgx: no version of go.dev satisfies "^1.27.1" AND is published for
linux/x86-64 (available: 1.27.1 1.26.8 1.26.5 … (11 in all);
satisfy but not published here: 1.27.1); asked for by ^1.27.1 (requested)
```

**"Satisfy but not published here"** is the distinction that matters: the
version exists as a tag and no bottle carries it for that platform. Before
the flag, `--graph` resolved for the host and said so about everywhere.

⛔ **`PKGX_CATALOG` does not redirect `--graph`.** `ls`, `search` and `why`
read the file that variable names. `--graph` resolves against the
**registry** and never opens it — so it refuses rather than answering about
the wrong machine:

```console
$ PKGX_CATALOG=~/.pkgx/catalog/linux-x86-64.json pkgx --graph +go.dev
pkgx: PKGX_CATALOG names …/linux-x86-64.json, and --graph does not read a
catalogue — it resolves against the registry. To ask about another platform:
--graph --platform os/arch
```

It used to answer the **host's** resolution there, with exit 0 and no sign
the file was never opened ([go-pkgx/pkgx#110](https://github.com/go-pkgx/pkgx/issues/110)).
A catalogue records what is *published*; `--graph` resolves what *would
satisfy* a constraint, including versions no bottle carries. Those are
different questions, and the "satisfy but not published here" line above
depends on telling them apart.

The price is that the index goes stale and nothing tells you by magic, so
every command that reads it says how old it is:

```console
$ pkgs catalog
/home/you/.pkgx/catalog/linux-x86-64.json
1908 project(s), 986 with dependencies, 2 hour(s) old
```

## Shipping a catalogue with the image {#shipping-a-catalogue-with-the-image}

`PKGX_CATALOG=<file>` reads one from somewhere else entirely — for an
air-gapped image, or to inspect a catalogue before publishing it. An image
that ships one can also simply drop the file at the path `pkgs catalog`
prints, and never fetch at all.

Set and unreadable is a **refusal**, not a quiet fall back to the local
store: somebody who names a file means that file, and answering from
somewhere else under a header blaming the registry would be three wrong
things in one line.

## When there is none {#when-there-is-none}

`pkgs ls` falls back to **what is installed**, and says so in its header —
with the sentence that fits the case it is actually in:

```console
$ pkgs ls                      # never fetched one
pkgs: showing what is INSTALLED — no catalogue yet; run: pkgs catalog update

$ pkgs ls                      # one is there and will not parse
pkgs: showing what is INSTALLED — /home/you/.pkgx/catalog/linux-x86-64.json could not be read
```

Those two are not the same state, and "run `pkgs catalog update`" is the fix
for the first and no help at all for the second. Sending somebody to a
command that cannot work is worse than saying nothing.

Offline, and in a fresh scratch image before the first fetch, the fallback
is not a lesser answer: it is the only true one available, and it is often
the one wanted, because completing onto something already installed costs
nothing to run.

The two lists are **never merged**. "Available" about a mix of a registry and
a local store is a word with no meaning, and a header cannot be honest about
a list with two sources.
