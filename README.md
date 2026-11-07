# go-zpkg docs

Documentation site for the [go-zpkg](https://github.com/go-zpkg) organization —
a package manager in pure, `CGO_ENABLED=0` Go. Built with Hugo and the
`tannevaled/hextra` theme fork (`hugo.yaml`, `themes/hextra`), and served at
<https://go-zpkg.github.io/docs/>.


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

BSD-3-Clause © the go-zpkg/docs authors.
