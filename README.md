# chicken-toolchain

Installs Chicken into a GitHub runner using a precompiled binary package from [ursetto/chicken-versions](https://github.com/ursetto/chicken-versions).

We currently support 5.3.0, 5.2.0, 5.1.0, 5.0.0 and 4.13.0 on Ubuntu 22.04 and 20.04 runners.

## Basic usage

Here's an example job to test an egg on Chicken 5.3.0 on `ubuntu-latest` with `chicken-install -test`. Place this in your egg's repository, in a file such as `.github/workflows/test.yml`.

```yaml
name: Test egg
on:
  - push
  - pull_request
  - workflow_dispatch
jobs:
  test:
    strategy:
      matrix:
        os:
          - ubuntu-latest
        chicken-version:
          - 5.3.0
    runs-on: ${{ matrix.os }}
    name: Test egg
    steps:
      - uses: actions/checkout@v3
      - uses: ursetto/chicken-toolchain@v0
        id: chicken
        with:
          chicken-version: ${{ matrix.chicken-version }}
      - run: chicken-install -test
```

The matrix strategy is not required when testing on a single OS and Chicken version, but having the boilerplate there makes it easier to add multiple versions later.

## Action inputs

- `chicken-version`: The exact version of Chicken to install. We currently support "4.13.0" through "5.3.0". Version constraints and aliases such as "5" are not supported.
- `update-environment`: If "true", updates your PATH to include the newly installed Chicken binaries. Defaults to "true".

## Action outputs

- `chicken-version`: The exact version of Chicken actually installed. This will always match the `chicken-version` input at the moment, but we might add input version aliases such as `"5"` in the future.
- `chicken-prefix`: The full path of the Chicken installation prefix. Binaries will be available under the `bin/` directory; for example `${{ steps.chicken.outputs.chicken-prefix }}/bin/csi`. On an Ubuntu runner, the prefix is generally `/home/runner/.local/chicken/<version>`, but avoid hardcoding this and use the output prefix or the PATH.

## Egg caching

This action does not include built-in [caching of install dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows), simply because I haven't decided on a nice interface for it.

In the meantime, you can cache eggs yourself. A simple example that you can add *before* running `chicken-install -test`:

```yaml
- name: Cache Chicken eggs
  id: cache-eggs
  uses: actions/cache@v3
  with:
    path: ~/.cache/chicken-install
    # Include chicken and OS version in the key (if building multiple versions)
    key: chicken-eggs-${{ matrix.chicken-version }}-${{ matrix.os }}-${{ hashFiles('*.egg') }}
```

All egg dependencies will be built in `~/.cache/chicken-install` during `chicken-install -test` on Chicken 5. The `cache` action will transparently save this directory at the end of the run, and restore it on next run. Note: due to [cache isolation](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache), feature branches can't see each others' caches, but everyone can see their parent's cache (usually `main`).

See [ursetto/sql-de-lite](https://github.com/ursetto/sql-de-lite/.github/workflows/test.yml) for a full example. This also happens to hash only the list of dependencies in the cache key, not the entire `.egg` file.