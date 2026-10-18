# go-pkgx docs

Documentation site for the [go-pkgx](https://github.com/go-pkgx) organization —
the pkgx world in pure, `CGO_ENABLED=0` Go. Built with MkDocs Material,
versioned with [mike](https://github.com/jimporter/mike), and served at
<https://go-pkgx.github.io/docs/>.

Covers running packages on `FROM scratch`, the signed & attested package
registry (`ghcr.io/go-pkgx/packages`), the supply-chain model, build isolation,
and Windows packages.

## Local preview

```bash
git submodule update --init --recursive   # themes/hextra, pinned at a release tag
hugo server                               # http://localhost:1313
```

Pushing to `main` deploys the `latest` version to the `gh-pages` branch via
GitHub Actions; GitHub Pages serves it under `/docs/`.

BSD-3-Clause © the go-pkgx/docs authors.
