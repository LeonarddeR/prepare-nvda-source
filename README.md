# prepare-nvda-source

A composite GitHub Action that produces a **built, runnable NVDA source clone** for NVDA add-on
CI. It checks out [`nvaccess/nvda`](https://github.com/nvaccess/nvda) at a given ref (with
submodules), installs the matching Python, installs the required Visual Studio components, and runs
`scons source`. The result is a `nvda/` directory with a compiled `source/` tree and a populated
`nvda/.venv` — i.e. an NVDA checkout you can import from and run tests against.

It exists so any NVDA add-on project can reuse a single, tested build step instead of copying the
same workflow into every repository.

## Requirements

- A **Windows** runner with a supported Visual Studio toolset installed. All current NVDA releases
  (2025.x, 2026.x) build with **Visual Studio 2022** (MSVC v143), which ships on the standard
  `windows-2022` and `windows-2025` GitHub-hosted images. By default the action adds the components
  from NVDA's `.vsconfig` to the installed VS at runtime, so a stock image works out of the box.
  See [Visual Studio & choosing a runner](#visual-studio--choosing-a-runner) for VS 2026 / custom
  runners.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | yes | — | Python version to install (e.g. `3.11`, `3.13.12`). |
| `python-arch` | yes | — | Python architecture: `x86` or `x64`. Must match the target NVDA build. |
| `nvda-ref` | yes | — | Git ref of `nvaccess/nvda` to build (tag or branch, e.g. `release-2026.1`). |
| `github-token` | yes | — | Token used to resolve `nvda-ref` to a commit SHA for a precise cache key. Pass `${{ github.token }}`. |
| `scons-args` | no | `-j2` | Extra arguments appended to `scons source` (e.g. `-j2`, `version=...`). |
| `install-vs-components` | no | `true` | Install NVDA's `.vsconfig` VS components at runtime. Set `false` on images that already ship the required VS toolset (e.g. a custom VS 2026 image) to skip the install. |
| `vs-version` | no | `` | Major VS version to select when installing components (`17` = VS 2022, `18` = VS 2026). Empty = latest installed. |

### 32-bit Python

When `python-arch` is `x64`, a secondary 32-bit (x86) Python is installed automatically (without
becoming the primary interpreter). NVDA's x64 builds contain 32-bit components, so building them
requires a matching 32-bit interpreter. For `x86` builds the primary interpreter is already 32-bit,
so nothing extra is installed. This mirrors what `nvaccess/nvda` does in its own build job.

## Outputs

| Output | Description |
|--------|-------------|
| `nvda-path` | Path to the built NVDA tree relative to the workspace (`nvda`). |
| `cache-hit` | `true` when the NVDA build was restored from cache (checkout/build skipped). |

## Caching

The whole `nvda/` tree (including the compiled `source/` and `.venv`) is cached with
`actions/cache`, keyed by `<os>-nvda-<python-version>-<python-arch>-<nvda-sha>`. On a cache hit,
checkout, VS-component install, and `scons source` are all skipped and the ready-to-use tree is
restored directly.

On a cache **miss**, SCons's detected MSVC environment is additionally cached (via
`SCONS_CACHE_MSVC_CONFIG`), keyed by the runner OS, image version, VS version, and a hash of NVDA's
`.vsconfig`. This skips the repeated MSVC toolchain detection on subsequent cold builds.

## Visual Studio & choosing a runner

NVDA's `.vsconfig` lists the required *components*, but not the VS *product version* — that comes
from whichever Visual Studio is installed on your runner. There are two ways to satisfy it:

- **Standard image + install components (default).** On `windows-2022` / `windows-2025`, keep
  `install-vs-components: true`. The action finds the installed VS with `vswhere` (any edition) and
  adds the `.vsconfig` components. This covers every current NVDA release (all build with VS 2022).
  On a multi-VS image, set `vs-version` (e.g. `17`) to pick a specific toolset.
- **Pre-baked image + skip install.** If your runner already ships the required toolset — e.g. a
  custom Visual Studio 2026 image for building `master`/frontier NVDA — set
  `install-vs-components: false` and target that runner. The action then behaves like
  `nvaccess/nvda`'s own build and does no runtime install.

Note on VS 2026: `nvaccess/nvda` builds the newest NVDA on its own **org-private** `windows-2025-vs2026`
runner, which you cannot reference from another account. To build against VS 2026 you must provide
your own runner (a self-hosted machine or a larger runner with a custom image) and use
`install-vs-components: false`, or wait until a standard GitHub image ships VS 2026. Runtime install
of VS 2026 on a stock image is not supported here — `vs_installer modify` only adds components to an
already-installed VS product; it cannot upgrade VS 2022 to VS 2026.

### Forks

Prefer a **standard** runner (`windows-2022` / `windows-2025`) where you can. Standard runners exist
in every account, so a consumer's own CI, forks of it, and fork PRs all build. A **custom** runner
(the `install-vs-components: false` path) only exists in the account that configured it: PRs *into*
that repo still build (they run on the base repo's runners), but a fork's own push-CI cannot use it.

## Usage

Keep your NVDA version matrix in your own repo (this action does not ship one). A typical setup
looks like this:

`.github/nvda-versions.json`:

```json
{
  "versions": [
    { "name": "2025.3", "python": "3.11",    "arch": "x86", "ref": "release-2025.3.3" },
    { "name": "2026.1", "python": "3.13.12", "arch": "x64", "ref": "release-2026.1" }
  ]
}
```

`.github/workflows/ci.yaml`:

```yaml
jobs:
  load-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v7
      - id: set-matrix
        run: echo "matrix=$(cat .github/nvda-versions.json | jq -c)" >> $GITHUB_OUTPUT

  test:
    needs: load-matrix
    runs-on: windows-2022
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.load-matrix.outputs.matrix).versions }}
    steps:
      - name: Clone add-on
        uses: actions/checkout@v7
        with:
          path: add-on

      - name: Prepare NVDA source
        uses: bramd/prepare-nvda-source@v1
        with:
          python-version: ${{ matrix.python }}
          python-arch: ${{ matrix.arch }}
          nvda-ref: ${{ matrix.ref }}
          github-token: ${{ github.token }}

      - name: Run unittests
        shell: cmd
        run: |
          cd add-on
          set PYTHONPATH=../nvda/source;../nvda/miscDeps/python
          ..\nvda\.venv\scripts\python.exe -m unittest discover -s tests -t .
```

The add-on is checked out to `add-on/` and NVDA lands as a sibling `nvda/`, so tests reference
`../nvda/source`, `../nvda/miscDeps/python`, and `../nvda/.venv`.

## Versioning

Pin the major tag: `uses: bramd/prepare-nvda-source@v1`. The moving `v1` tag tracks the latest
`v1.x` release. See [`CHANGELOG.md`](CHANGELOG.md).

## License

GPL v2. See [`LICENSE`](LICENSE).
