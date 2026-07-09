# Changelog

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
