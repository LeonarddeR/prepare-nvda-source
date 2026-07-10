# Changelog

## v1.1.2

- Simplify the VS component install: run `vs_installer.exe` directly instead of via a `cmd.exe`
  wrapper, locate the installer via `${ProgramFiles(x86)}` rather than a hardcoded drive path, and
  express the deliberate double-run as a loop documenting why (the first pass may only self-update
  the installer).
- Remove the redundant `nvda-ref` output (it only echoed the caller's own input). The `nvda-path`
  and `cache-hit` outputs are unchanged.

## v1.1.1

- Install uv after the NVDA checkout/restore (with `cache-dependency-glob: nvda/uv.lock`) so
  `setup-uv` can key its cache on NVDA's lockfile, removing the "cache will never get invalidated"
  warning.

## v1.1.0

- Add `install-vs-components` input (default `true`) to skip the runtime VS component install on
  images that already ship the required Visual Studio toolset (e.g. a custom VS 2026 image),
  matching `nvaccess/nvda`'s own build.
- Add `vs-version` input to select a specific major VS version (`17` = VS 2022, `18` = VS 2026)
  when multiple are installed.
- Locate Visual Studio with `vswhere` instead of a hardcoded `2022\Enterprise` path, so the
  install works across Community/Enterprise/BuildTools and different VS versions.
- Include the VS version in the SCons MSVC config cache key.
- Document the standard-vs-custom-runner and fork tradeoffs.

## v1.0.0

Initial release of `prepare-nvda-source`, a composite action that builds a runnable NVDA source
tree for add-on CI.

- Checkout `nvaccess/nvda` at a given ref (recursive submodules), install matching Python
  (version + arch), install the VS 2022 components from NVDA's `.vsconfig`, and run `scons source`
  to produce a runnable NVDA clone with `nvda/.venv` and a compiled `source/` tree.
- Cache the built `nvda/` tree keyed by `<os>-nvda-<python-version>-<python-arch>-<nvda-sha>`.
- Cache SCons's detected MSVC environment (`SCONS_CACHE_MSVC_CONFIG`) on cold builds, keyed by
  runner OS, image version, and a hash of NVDA's `.vsconfig`, to skip repeated MSVC toolchain
  detection.
- Automatically install a secondary 32-bit (x86) Python for `x64` builds (NVDA's x64 builds
  contain 32-bit components); `x86` builds already have a 32-bit primary interpreter.
- Modeled on `nvaccess/nvda`'s current `testAndPublish.yml` build job: `actions/checkout@v7`,
  `actions/cache@v5`, dual x64/x86 Python, `astral-sh/setup-uv@v7`, MSVC config cache, and a
  `scons-args` passthrough input (default `-j2`).
