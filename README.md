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
  Frontier NVDA (`master`) builds with **Visual Studio 2026**, which ships pre-installed on the
  standard `windows-2025-vs2026` image — see
  [Visual Studio & choosing a runner](#visual-studio--choosing-a-runner).

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `` | Python version to install (e.g. `3.13.12`). Empty = auto-detect from NVDA's `.python-versions` (NVDA ≥ 2025.3). See [Automatic Python detection](#automatic-python-detection). |
| `python-arch` | no | `` | Python architecture: `x86` or `x64`. Empty = auto-detect from NVDA's `.python-versions` (NVDA ≥ 2025.3). Must match the target NVDA build. |
| `nvda-ref` | yes | — | Git ref of `nvaccess/nvda` to build (tag or branch, e.g. `release-2026.1`). |
| `github-token` | yes | — | Token used to resolve `nvda-ref` to a commit SHA for a precise cache key. Pass `${{ github.token }}`. |
| `scons-args` | no | `-j2` | Extra arguments appended to `scons source` (e.g. `-j2`, `version=...`). |
| `install-vs-components` | no | `true` | Install NVDA's `.vsconfig` VS components at runtime. Set `false` on images that already ship the required VS toolset (e.g. the `windows-2025-vs2026` image) to skip the install. |
| `vs-version` | no | `` | Major VS version to select when installing components (`17` = VS 2022, `18` = VS 2026). Empty = latest installed. |

### Automatic Python detection

If you leave `python-version` and/or `python-arch` empty, the action reads NVDA's root
`.python-versions` file at the target ref and installs the interpreter NVDA itself pins. NVDA's uv is
`python-preference = only-system`, so the build only finds its interpreter when the *exact* patch
NVDA declares is present — auto-detection keeps you in sync automatically instead of hard-coding a
version that drifts release-to-release.

Details and caveats:

- **Source of truth.** Each line of `.python-versions` is `cpython-<X.Y.Z>-windows-<arch>-none`. The
  action uses the **first** line as the primary interpreter and maps `x86_64` → `x64`, `x86` → `x86`.
- **NVDA ≥ 2025.3 only.** The file was introduced in NVDA 2025.3. For older refs, auto-detection
  fails with a clear error — pass `python-version` and `python-arch` explicitly for those.
- **Explicit wins.** Any value you pass overrides the detected one (you can auto-detect just the
  version and pin the arch, or vice versa).
- **Brand-new patches.** Detection installs the exact patch via `actions/setup-python`. In the rare
  window where NVDA pins a patch newer than `actions/setup-python`'s manifest, pin `python-version`
  explicitly until the manifest catches up.

### 32-bit Python

When the (detected or explicit) architecture is `x64`, a secondary 32-bit (x86) Python is installed
automatically (without becoming the primary interpreter). NVDA's x64 builds contain 32-bit
components, so building them requires a matching 32-bit interpreter. For `x86` builds the primary
interpreter is already 32-bit, so nothing extra is installed. This mirrors what `nvaccess/nvda` does
in its own build job.

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
- **Pre-baked image + skip install.** If your runner already ships the required toolset, set
  `install-vs-components: false` and target that runner. The action then behaves like
  `nvaccess/nvda`'s own build and does no runtime install. This is the right setting for
  `windows-2025-vs2026` (see below).

Building against VS 2026 (`master`/frontier NVDA): `nvaccess/nvda` builds the newest NVDA on the
**standard, generally-available** `windows-2025-vs2026` GitHub-hosted image, which ships Visual
Studio 2026 Enterprise pre-installed. It is a normal runner label available in every account, so you
can use it directly:

```yaml
runs-on: windows-2025-vs2026
# ...
      - uses: bramd/prepare-nvda-source@v1
        with:
          # ...
          install-vs-components: false   # VS 2026 + NVDA's components already on the image
```

Do **not** try to build VS 2026 on a stock `windows-2022` image with `install-vs-components: true`:
`vs_installer modify` only adds components to an already-installed VS product; it cannot upgrade VS
2022 to VS 2026. Pick the `windows-2025-vs2026` runner instead.

### Forks

Every runner label this action targets (`windows-2022`, `windows-2025`, `windows-2025-vs2026`) is a
standard GitHub-hosted image available in every account, so a consumer's own CI, forks of it, and
fork PRs all build without any account-specific setup.

## Usage

Keep your NVDA version matrix in your own repo (this action does not ship one). A typical setup
looks like this:

`.github/nvda-versions.json`:

```json
{
  "versions": [
    { "name": "2025.3", "ref": "release-2025.3.3" },
    { "name": "2026.1", "ref": "release-2026.1" }
  ]
}
```

Python version and architecture are auto-detected per ref (see
[Automatic Python detection](#automatic-python-detection)), so the matrix only needs the NVDA ref.
Add `python`/`arch` entries and pass them through if you target NVDA < 2025.3 or want to pin a
specific interpreter.

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
          nvda-ref: ${{ matrix.ref }}
          github-token: ${{ github.token }}
          # python-version / python-arch omitted -> auto-detected from NVDA's .python-versions

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
