# PARALLEL_BUILD_ATOM.md

## Why this branch exists

`components.yaml` lists each component's `remote:` as a **relative** path (e.g.
`../ESMA_env.git`). `mepo` resolves those against this repository's `origin`
URL. Upstream, origin is `github.com/GEOS-ESM/GEOSgcm`, so `../ESMA_env.git`
resolves correctly to `github.com/GEOS-ESM/ESMA_env.git`.

In this fork, origin is `github.com/dsg-atom/GEOSgcm`, so the same relative
paths resolve to `github.com/dsg-atom/ESMA_env.git` — repositories that don't
exist — and `mepo clone` fails with `Repository not found`.

On the `components-absolute-paths` branch, every `remote:` in `components.yaml`
has been rewritten to its **absolute** URL
(`https://github.com/GEOS-ESM/<repo>.git`), so components resolve regardless of
the fork's origin.

## If `mepo` still points at `dsg-atom`

`mepo` caches its parsed configuration in a `.mepo` state directory the first
time it runs. If you attempted a clone **before** checking out this branch, that
cache still holds the old `dsg-atom`-resolved URLs, and fixing `components.yaml`
afterward will not update it. You'll still see:

```
fatal: repository 'https://github.com/dsg-atom/ESMA_env.git/' not found
```

Clear the stale state, then run the build again:

```sh
rm -rf .mepo .mepo.lock
```
