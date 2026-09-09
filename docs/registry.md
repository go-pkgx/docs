# Package registry

[`ghcr.io/go-pkgx/packages`](https://github.com/go-pkgx/packages) is a pure-Go
package factory: it builds [pkgx pantry](https://github.com/pkgxdev/pantry)
recipes and publishes **signed, attested packages** for **every platform** through
**two channels**.

## Two channels, every platform

Every platform — **linux/x86-64**, **linux/aarch64**, **darwin/aarch64**,
**darwin/x86-64** and **windows/x86-64** — publishes to **both**:

- the **signed OCI registry** `ghcr.io/go-pkgx/packages` (each package carrying an
  SBOM, provenance and signature as OCI referrers), and
- a **GitHub Pages pkgx-dist mirror** at
  [`https://go-pkgx.github.io/packages`](https://go-pkgx.github.io/packages),
  carrying all platforms side by side (`<project>/<os>/<arch>/…`).

## The factory

The [`go-pkgx/packages`](https://github.com/go-pkgx/packages) repo is the factory.
GitHub Actions workflows build each recipe with
[`bk`](https://github.com/go-pkgx/bk) — the CGO-free re-implementation of brewkit
— and publish the result to both channels.

- **Factories:** `build.yml` (linux), `darwin.yml` (macOS), `windows.yml` (Go) +
  `windows-rust.yml` (Rust) + `windows-proof.yml` (e2e), each publishing to the OCI
  registry and uploading a pkgx dist tree; a single `pages.yml` aggregator unions
  the latest run of every build into the combined Pages mirror.
- **Schedule:** each factory runs on a schedule (plus manual `workflow_dispatch`).
- **Auth:** the workflow's **native `GITHUB_TOKEN`** (`permissions.packages: write`)
  — no long-lived PAT to manage or rotate.
- **Ordering:** requested projects are expanded to their **topologically-ordered
  runtime-dependency closure**, so every dependency is built before its dependents.
- **Idempotent:** any `(project, version, platform)` already in ghcr is **skipped**,
  so shared dependencies build once and the catalog grows progressively.

Per-recipe failures are logged, never fatal; the recipe list is grown outward from
dependency-free leaves toward the full pantry.

## What is published

Packages are ordinary OCI artifacts — each with a signature, an SBOM, and a
provenance statement attached as [referrers](supply-chain.md). Because the
factories keep adding recipes, treat the published set as a moving target — it
grows continuously. Example packages published at the time of writing (a snapshot,
not the full catalog):

| project | version |
| --- | --- |
| `zlib.net` | 1.3.2 |
| `tukaani.org/xz` | 5.8.3 |
| `lz4.org` | 1.10.0 |
| `gnu.org/tar` | 1.35 |
| `sourceware.org/bzip2` | 1.0.8 |

## Trees the pantry does not have

Most of the catalog is a pkgx pantry recipe built here. These are projects
upstream does not carry at all — the HPC and GPU floor an MPI actually needs,
which the pantry stops just above:

| project | | |
| --- | --- | --- |
| `openucx.org` | 1.22.0 | the transport OpenMPI, MPICH and OpenSHMEM reach shared memory and RDMA through — built `--with-verbs`, so `libuct_ib`, `libuct_ib_mlx5` and `libuct_ib_efa` are there rather than TCP alone |
| `github.com/linux-rdma/rdma-core` | 65.0 | `libibverbs`, which is what makes the line above more than a configure flag |
| `github.com/ROCm/ROCR-Runtime` | 7.2.4 | the HSA runtime an AMD GPU is driven through, **built from NCSA source**; linux/x86-64 only, because upstream's `utils.h` calls an x86 intrinsic unguarded |
| `nvidia.com/cuda-cudart` | 13.3.1 | the CUDA runtime, from NVIDIA's redistributable set, verified against the sha256 in their own manifest |
| `kernel.org/linux` | 6.19.14 | the microVM kernel, with `INFINIBAND_USER_ACCESS`, `RDMA_RXE`, `RDMA_SIW`, `MLX5_*`, `VFIO_*`, hugepages and `PCI_P2PDMA` — the devices `libibverbs` opens, and GPUDirect |
| `github.com/containers/crun` | 1.29.1 | the runtime that closes the microVM boot loop |

The two GPU rows differ in kind, and the recipes say so: ROCm is source we
compile, CUDA is an archive we fetch under a licence that names which files may
travel.

Nothing here needs a GPU or a fabric to **build**. What a build machine can
honestly check is that a runtime links, loads and answers — `hsa_init()`
returning "out of resources" with no `/dev/kfd` is a pass — and that a transport
module was compiled at all, since a silently declined `--with-verbs` looks
exactly like a missing `libuct_ib.so`.

## Consuming

Point the go-pkgx tools at the registry and verify against the pinned key:

```sh
PKGX_DIST=oci://ghcr.io/go-pkgx/packages PKGX_VERIFY=1 pkgm install lz4.org
```

`PKGX_VERIFY=1` is fail-closed: an unsigned or badly-signed package is refused, not
installed. See [supply chain](supply-chain.md) for the verification model.

Or consume the same packages from the GitHub Pages pkgx-dist mirror:

```sh
PKGX_DIST=https://go-pkgx.github.io/packages pkgm install lz4.org
```

Packages are OCI artifacts, so any OCI client can also pull them directly:

```sh
docker pull ghcr.io/go-pkgx/packages/lz4.org:1.10.0
```
