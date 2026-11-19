# pydantic: test dependencies on CPython 3.14 across three platforms

[pydantic](https://github.com/pydantic/pydantic) is a data validation library built on its own Rust core. This case study checks its locked **test environment**, not just its runtime dependencies, at revision [`328b88b52694a64ecd05f3acc92f2b6a160bc2ee`](https://github.com/pydantic/pydantic/tree/328b88b52694a64ecd05f3acc92f2b6a160bc2ee). The scan used **wheelgap 0.13.0** and package files available on **19 November 2026** (US Eastern time).

The result is a test environment whose every package installs from a wheel: 56 packages across the three targets, no source builds at all.

## Targets and selection

The pinned [CI test matrix](https://github.com/pydantic/pydantic/blob/328b88b52694a64ecd05f3acc92f2b6a160bc2ee/.github/workflows/ci.yml#L866-L891) runs `uv run pytest` on `ubuntu-latest`, `macos-latest` and `windows-latest` for CPython 3.11 through 3.15 plus 3.15t. This study checks the 3.14 jobs on:

- `linux-x86_64` and `win-amd64`, the x64 Linux and Windows runners.
- `macos-arm64`, the Apple-silicon runner. The workflow's `macos-latest` label alone doesn't establish the runner's architecture; across GitHub-hosted images it is ARM today, and this study says so rather than assuming it for every reader.

Linux uses wheelgap's default glibc floor **2.36**; it doesn't measure the runner's installed libc, and it makes no claim about other Linux distributions.

The CI job installs the workspace with `uv sync --all-packages --group testing-extra --all-extras` ([ci.yml:894-895](https://github.com/pydantic/pydantic/blob/328b88b52694a64ecd05f3acc92f2b6a160bc2ee/.github/workflows/ci.yml#L894-L895)), so the wheelgap command selects the same: every project extra, plus the `testing-extra` group. That includes the test tools and their Python dependencies, and should not be read as an audit of pydantic's runtime footprint.

## Reproduce the scan

In an empty folder, download the pinned lockfile and check it:

```bash
curl -fL https://raw.githubusercontent.com/pydantic/pydantic/328b88b52694a64ecd05f3acc92f2b6a160bc2ee/uv.lock -o uv.lock
uvx --from wheelgap==0.13.0 python -m wheelgap uv.lock \
  -p 3.14 -t linux-x86_64,macos-arm64,win-amd64 \
  --group testing-extra --all-extras \
  --explain --suggest --no-cache
```

Output:

```text
wheelgap 0.13.0 · uv.lock · 56 packages · Python 3.14

skipped 2 non-index packages: pydantic (editable), pydantic-core (editable)
✓ All 56 packages have a wheel for every one of 3 targets.
```

The command exits **0**, since it reports no gaps. The two skipped packages are the workspace's own `pydantic` and `pydantic-core`, which the checkout builds from source; `uv sync --all-packages` does the same.

## The runtime dependencies on their own

`--no-dev` narrows the check to what pydantic installs for a user, which is three registry packages:

```bash
uvx --from wheelgap==0.13.0 python -m wheelgap uv.lock \
  -p 3.14 -t linux-x86_64,macos-arm64,win-amd64 --no-dev --explain --suggest --no-cache
```

```text
wheelgap 0.13.0 · uv.lock · 3 packages · Python 3.14

skipped 2 non-index packages: pydantic (editable), pydantic-core (editable)
✓ All 3 packages have a wheel for every one of 3 targets.
```

Both runs exit **0**, so there is nothing to fix in the lockfile: no gap here needs a newer release, a different package, a source build in CI, or a dropped platform.

## What a whole-lockfile scan also reports

The lockfile has 156 packages across every dependency group (`all`), including the documentation toolchain. Checking all of it with the same targets reports two gaps, and they are worth naming because they show what the flags change:

```bash
uvx --from wheelgap==0.13.0 python -m wheelgap uv.lock \
  -p 3.14 -t linux-x86_64,macos-arm64,win-amd64 --explain --suggest --no-cache
```

```text
package         version  linux-x86_64  macos-arm64  win-amd64
mkdocs-exclude  1.0.2    ● source      ● source     ● source
watchdog        6.0.0    ✓             ● source     ✓

154 packages with wheels for every target hidden (--all to show)
```

- **`mkdocs-exclude` 1.0.2** publishes an sdist only, so three targets report `source` with `no wheels published for this version`. `--suggest` names 1.0.0 as the last release with a wheel.
- **`watchdog` 6.0.0** has `cp311`–`cp313` macOS wheels but no `cp314` one, so `macos-arm64` reports `source`; `--suggest` names 7.0.0 as the release that has it. Linux and Windows get wheels (`py3-none-manylinux2014_x86_64`, `py3-none-win_amd64`).

Neither package is in what the CI job installs: `mkdocs-exclude` is in the `docs` group and `watchdog` reaches the tree through `mkdocs`. That is why the `--group testing-extra --all-extras` run above is the one that answers "does the test environment install cleanly", and the whole-lockfile run answers a different question.

## Independent pip checks

The two whole-lockfile gaps were checked against public PyPI with pip **26.2.1**, `--no-deps --only-binary=:all: --python-version 3.14 --implementation cp --abi cp314`:

```bash
python -m pip --isolated download --no-deps --only-binary=:all: \
  --python-version 3.14 --implementation cp --abi cp314 \
  --platform macosx_14_0_arm64 --dest wheelhouse "watchdog==6.0.0"
python -m pip --isolated download --no-deps --only-binary=:all: \
  --python-version 3.14 --implementation cp --abi cp314 \
  --platform macosx_14_0_arm64 --dest wheelhouse "mkdocs-exclude==1.0.2"
```

Both returned **No matching distribution found**, which agrees with wheelgap. The Linux and Windows cells were checked the same way with `--platform win_amd64` and with every `manylinux_2_*` tag from 2.36 down to 2.5 plus `manylinux2014`, `manylinux2010` and `manylinux1` for `x86_64`: `watchdog==6.0.0` downloaded `py3-none-manylinux2014_x86_64.whl` and `py3-none-win_amd64.whl`, and `mkdocs-exclude==1.0.2` found no distribution on either.

The releases `--suggest` names were checked too: `watchdog==7.0.0` downloaded `cp314-cp314-macosx_11_0_arm64.whl`, and `mkdocs-exclude==1.0.0` downloaded `py3-none-any.whl`.

None of this inspects or tests pydantic's own code: both checks read published package files, and neither runs a dependency's build.
