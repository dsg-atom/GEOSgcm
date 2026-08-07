# Building this fork

## Why this branch exists

`components.yaml` normally lists each component's `remote:` as a **relative**
path, e.g. `../ESMA_env.git`. `mepo` resolves those relative paths against this
repository's `origin` URL. In the upstream repo that origin is
`github.com/GEOS-ESM/GEOSgcm`, so `../ESMA_env.git` correctly resolves to
`github.com/GEOS-ESM/ESMA_env.git`.

In this fork the origin is `github.com/dsg-atom/GEOSgcm`, so the same relative
paths resolve to `github.com/dsg-atom/ESMA_env.git` — repositories that do not
exist. `mepo clone` (invoked by `parallel_build.csh`) then fails with:

```
remote: Repository not found.
fatal: repository 'https://github.com/dsg-atom/ESMA_env.git/' not found
```

On the `components-absolute-paths` branch, every `remote:` in `components.yaml`
has been rewritten to its **absolute** URL
(`https://github.com/GEOS-ESM/<repo>.git`) so the components can be located
regardless of the fork's origin.

## Getting the branch on the build host

Run these in the clone on the host (the top-level `GEOSgcm` directory that has
`dsg-atom/GEOSgcm` as `origin`):

```sh
git fetch origin
git checkout components-absolute-paths

# Confirm the fix is present — should print the GEOS-ESM URL:
grep -A1 '^env:' components.yaml
```

## Building

```sh
./parallel_build.csh
```

## If `mepo clone` still points at `dsg-atom`

`mepo` caches its parsed configuration in a `.mepo` state directory the first
time it runs. If you attempted a clone **before** checking out this branch, that
cache still holds the old `dsg-atom`-resolved URLs, and fixing `components.yaml`
afterward will not update it. Clear the stale state and re-run:

```sh
rm -rf .mepo .mepo.lock
./parallel_build.csh
```
