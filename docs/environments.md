# Environments, and HPC

A module system does two things: it resolves what a package needs, and it edits
your shell. `pkgx +a +b` already did the first. `pkge` does the second — a
front-end shaped like Lmod's, over pkgx's own resolution, with no Lua
interpreter and no modulefile tree.

It ships in the `pkgx` binary; there is nothing else to install.

```console
$ eval "$(pkgx env init)"          # in a profile
$ pkge load stedolan.github.io/jq
$ command -v jq
~/.pkgx/stedolan.github.io/jq/v1.8.2/bin/jq
$ pkge list
stedolan.github.io/jq
$ pkge unload stedolan.github.io/jq
$ command -v jq
/usr/bin/jq                        # back, exactly as it was
```

`pkge save <name>` writes the current set to
`$PKGX_DIR/collections/<name>`; `pkge restore <name>` purges and reloads it.
`pkge purge` unloads everything.

## Why unloading is exact

Loading never edits incrementally. It restores the environment it first saw and
recomposes from the whole spec set, so `unload b` after `load a b c` leaves
precisely what `load a c` would have — independent of the order things were
loaded in.

The state that makes this work lives in the environment, not in the shell, and
is read back by `pkgx`:

- `PKGE_SPECS` — the package set currently loaded.
- `PKGE_BASE` — the *original* value of every variable pkgx has touched,
  base64 of NUL-separated `NAME=value` records.

A variable's base is captured once, the first time something touches it: the
second load must not record the first load's output as the pristine value. A
variable that did not exist before is restored by being *unset*, not by being
set empty — an empty `PATH` and no `PATH` are different to every program that
reads one.

Lmod carries a module table for the same reason. This carries the base instead,
which is smaller and cannot disagree with itself.

`LOADEDMODULES` is maintained, in Lmod's own colon-separated spelling, because
job scripts read it — `if [[ $LOADEDMODULES == *openmpi* ]]` is in thousands of
them. `_LMFILES_` is deliberately **not** set: it names modulefiles, there are
none here, and an invented value is a lie a script could act on.

## Named environments

A site declares environments in HCL2 under `$PKGX_DIR/environments/`:

```hcl
# ~/.pkgx/environments/site.hcl2
env "cfd" {
  description = "the solver stack, as validated on 2026-09-01"
  packages    = ["openmpi.org@5", "hdf5.org", "python.org@3.12"]
}
```

An `env` block takes `description`, `packages`, and the environment edits
`prepend_path`, `append_path`, `setenv` and `unsetenv`.

```console
$ pkgx env avail
cfd                      the solver stack, as validated on 2026-09-01
hdf5                     HDF5 1.14.3

$ pkgx env show cfd
env "cfd"
  description  the solver stack, as validated on 2026-09-01
  declared in  /home/you/.pkgx/environments/site.hcl2
  package      openmpi.org@5
  package      hdf5.org
  package      python.org@3.12

$ pkge load cfd
```

An environment is not a lockfile: it names constraints, and the closure is
resolved at load time from the signed registry, so a login node and a container
agree.

## With an existing Lmod

Generate modulefiles and let the site's own Lmod load them. Nothing about
`module` changes, and conflicts, hierarchies and `spider` keep working, because
Lmod is still the one doing the work.

```console
$ pkgx --modulefile +openmpi.org@5 > $MODULEPATH_DIR/openmpi/5.0.8.lua
```

`pkgx --json +<pkg>...` emits the same composed environment as data, for a tool
that wants to compose it some other way.

## Without one

`pkgx env init --module` also defines `module` and `ml`
(`load`/`add`, `unload`/`rm`/`del`, `purge`, `list`, `avail`/`av`,
`show`/`display`/`whatis`, `save`, `restore`, `swap`/`switch`) — for a
container, a laptop, or a cluster built on this toolchain.

It **refuses** to install itself where an Lmod already exists:

```
pkgx: an existing module system was found; not defining module().
pkgx: publish modulefiles to MODULEPATH instead: pkgx --modulefile +pkg
```

Two implementations answering one command is how a support ticket becomes
unanswerable, and the site's own modulefiles are not ours to interpret.

## Converting a site's modulefiles

`pkgx env import` reads Lmod's Lua **and** Environment Modules' TCL, and writes
both into the same HCL2.

```console
$ pkgx env import /opt/modulefiles/hdf5
# imported from /opt/modulefiles/hdf5 by `pkgx env import`.
# Every statement in that file was recognised; nothing was skipped.
env "hdf5" {
  description = "HDF5 1.14.3"
  prepend_path = {
    LD_LIBRARY_PATH = ["/opt/hdf5/1.14.3/lib"]
    PATH = ["/opt/hdf5/1.14.3/bin"]
  }
  setenv = {
    HDF5_DIR = "/opt/hdf5/1.14.3"
  }
}
```

It is a **parser, not an evaluator**. An interpreter runs a modulefile and
silently skips what it does not implement; the result is not an error but an
environment that is subtly wrong, found later inside somebody's job. This
refuses anything it would have to guess at, names the line, and — the part that
matters — **writes nothing at all** if any file was refused, because a partial
conversion is the one outcome nobody can check.

```console
$ pkgx env import /opt/modulefiles/openmpi.lua /opt/modulefiles/hdf5
/opt/modulefiles/openmpi.lua: 3 statement(s) this converter will not guess at:
  line 3: not a plain call: this converter reads statements, it does not run Lua
    local root = "/opt/openmpi/5.0.8"
  line 4: argument is not a string literal (pathJoin(root, "bin")): its value depends on something we are not running
    prepend_path("PATH", pathJoin(root, "bin"))
  line 5: depends_on states a RELATIONSHIP between modules, not an environment change: decide it in the environment that replaces this one
    depends_on("hwloc")
pkgx: 1 of 2 modulefile(s) not converted — nothing written, because a PARTIAL conversion is the one outcome nobody can check
$ echo $?
1
```

`conflict`, `family` and `depends_on` are refused on purpose rather than left
for later: they state relationships *between* modules, which is a decision for
the environment that replaces them, not something a converter can infer.

## Two things no code here removes

- **MPI and the interconnect come from the host.** libfabric/UCX, Slurm's PMI
  and a vendor libmpi tied to a kernel driver cannot live in a hermetic tree.
  Bind-mount them, as Spack and Apptainer do.
- **Metadata storms.** Ten thousand ranks walking a shared `$PKGX_DIR` will melt
  a Lustre MDS. Materialise once into a squashfs or SIF and mount it read-only.
