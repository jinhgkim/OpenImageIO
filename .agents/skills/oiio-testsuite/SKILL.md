---
name: oiio-testsuite
description: Rules and conventions for OpenImageIO's end-to-end testsuite (testsuite/<name>/run.py, ref/ outputs, runtest.py helpers, registration in src/cmake/testing.cmake). Use when adding or changing a testsuite test, adding a regression fixture, updating or adding reference output, adjusting image diff thresholds, or diagnosing a testsuite failure locally or in CI.
---

# OIIO testsuite conventions

These are rules, not a procedure. The harness is `testsuite/runtest.py`;
read it when a rule here is not enough. `testsuite/TESTSUITE-README.md`
covers diagnosing CI failures and reference variants in more depth.

## Choosing where a test lives

- Testsuite tests run the built tools (`oiiotool`, `iinfo`, `idiff`,
  `maketx`, `iconvert`, `igrep`, `testtex`) or Python scripts and compare
  their outputs against `ref/`. C++ unit tests are separate:
  `src/lib*/<name>_test.cpp`, registered with `add_test(unit_<name> ...)` in
  that directory's `CMakeLists.txt`.
- Extend an existing test directory before creating a new one. A new
  regression case for a format goes in that format's `run.py`.
- Make a new directory when the test needs different registration (no
  external images, a different dependency gate, an env variant), e.g.
  `png-bomb` and `openexr-bomb` exist so they run without `oiio-images`.
- A bug fix or behavior change ships with a test that exercises it in the
  same PR (AGENTS.md change checklist). Test-only changes use the `test:`
  commit type.

## Test directory anatomy

- `testsuite/<name>/run.py`: required, with the standard copyright + SPDX
  header. It is `exec`'d inside `runtest.py`, so it appends to `command`
  and may set `outputs` (default `["out.txt"]`) and the knobs below.
- `ref/`: reference outputs. `src/`: inputs, fixtures, helper scripts.
  Optional `CMakeLists.txt` for a compiled helper (see `openexr-partialtile`).
- The test runs in `build/testsuite/<name>/`. `ref/` and `src/` are
  symlinked there (copied on Windows). Refer to inputs as `src/<file>`.
  The `data` symlink to the source dir is not created on Windows.
- External image repos are reached through `OIIO_TESTSUITE_IMAGEDIR`. They
  are separate repos (`oiio-images` is pinned to a `dev-X.Y` branch), so a
  PR here cannot add files to them. Prefer self-contained fixtures in `src/`.

## Registration (`src/cmake/testing.cmake`, `oiio_add_all_tests`)

- Every new directory must be added to an `oiio_add_tests(...)` call or it
  never runs.
- Gate on what the test actually needs, nothing more:
  - `ENABLEVAR ENABLE_<FMT>` for a format plugin that can be disabled.
  - `FOUNDVAR <Pkg>_FOUND` for an optional dependency.
  - `IMAGEDIR <dir>` (plus `URL`) only if it reads external images.
    Without it the test runs everywhere.
- Variants reuse the same directory via `SUFFIX` + `ENVIRONMENT`
  (`.batch`, `.core`, `.hwy`, `.nanobind`). A test added to a list that has
  variants (e.g. `all_texture_tests`, `all_openexr_tests`) gets them too.
- Python-driven tests must sit inside the `USE_PYTHON AND NOT
  BUILD_OIIOUTIL_ONLY AND NOT SANITIZE` block; Python tests do not run in
  sanitizer builds. `PYTHONPATH` to this build's bindings is set for every
  test already; do not add a per-test copy.

## Writing `run.py`

- Build commands with the helpers, not raw paths: `oiiotool()`,
  `run_commands("""...""")` (one shell command per line, `#` lines
  skipped), `info_command()`, `diff_command()`, `rw_command()`,
  `iconvert()`, `maketx_command()`, `testtex_command()`, `run_app()`.
  `run_app` resolves OIIO tool names to the built binaries. Use `pythonbin`
  for Python.
- `command` is split on `;` and each piece run separately, so a literal `;`
  inside a command's arguments breaks it.
- Default `redirect` captures stdout only. When an error message is the
  thing under test, set `redirect = ' >> out.txt 2>&1 '`, or the error
  never reaches `out.txt`.
- Expected nonzero exits: prefer per-call `failureok=True` on the commands
  that should fail. The global `failureok = 1` stops every command's exit
  status from failing the test, which can hide an unexpected crash.
- Use `info_command(..., safematch=True)` for files whose metadata varies
  between runs or versions (it hides `DateTime|Software|OriginatingProgram|
  ImageHistory`). `hash=True` (default) prints a pixel hash, the cheapest
  strong check that pixels decoded correctly.
- Generate simple inputs at test time with `oiiotool --pattern` / `--create`
  rather than committing images.
- Each case gets a short comment saying what it guards and why; regressions
  cite the issue or PR number. Match the existing comment density of the
  file.
- `redirect_push(file)` / `redirect_pop()` send a section of output to an
  extra file that is checked against its own ref.
- `# BEGIN-docs-<id>` / `# END-docs-<id>` markers inside `run_commands`
  blocks are `literalinclude`d by `src/doc/oiiotool.md`. Do not rename or
  delete them without updating the docs.
- Python test scripts live in `src/` and print results to stdout, which
  `run.py` redirects to `out.txt`.

## Fixtures

- Keep fixtures minimal. Fixtures in a format's `testsuite/<fmt>/src` (and
  dirs listed in `FORMAT_SOURCES` in `src/fuzz/populate_corpora.py`) become
  fuzz seeds automatically, capped at 5 MB per file; do not also copy them
  into `src/fuzz/corpora/`. A new dedicated fixture dir needs adding to
  `FORMAT_SOURCES` (see `docs/dev/fuzzing.md`).
- Crafted or malformed files: commit the fixture and the script that made
  it (`src/make-*.py`, e.g. `psd-empty-layers`, `jpeg-corrupt`), so it can
  be regenerated and reviewed. Generating at test time is also fine when it
  is fast and deterministic.
- For malformed-input fixture design (bombs, truncation, bad offsets), see
  the `oiio-input-security-audit` skill.

## Reference output

- Matching: an output passes if it matches `ref/<name>` or any
  `ref/<prefix>-*<ext>*` variant (any file at all if `anymatch = True`).
  `refdirlist` can add other tests' `ref/` dirs.
- Comparison by extension:
  - `.txt`: text diff, trailing whitespace ignored, CRLF normalized on
    Windows.
  - Extensions in `image_extensions` in `runtest.py`: `idiff` with the
    threshold knobs.
  - Anything else (including image formats missing from that list):
    byte-exact.
- Every entry in `outputs` needs a ref, or the test reports NO MATCH.
- Never regenerate refs blindly. Diff old vs new and confirm every change
  is intended by the code change before copying `build/testsuite/<name>/`
  outputs into `ref/`.
- Platform or dependency-version differences get a new, self-describing
  variant (`out-win.txt`, `out-osx.txt`, `text-freetype2.7.tif`,
  `out-libfoo3.4.txt`); never overwrite the existing ref, which would break
  the platforms it matches.
- Thresholds (`failthresh`, `failpercent`, `hardfail`, `allowfailures`,
  `failrelative`) are set per test in `run.py`, raised by the minimum
  needed, with a comment saying why (see `testsuite/raw/run.py`). They are
  doubled automatically when `CI` or `DEBUG` is set. If a small bump is not
  enough, add a variant ref instead.

## Running and diagnosing

- `ctest --test-dir build -R <pattern> --output-on-failure` or
  `make test TEST=<pattern>`. Outputs land in `build/testsuite/<name>/`,
  with `<out>.diff` beside any text mismatch.
- `runtest.py` pins `limits:imagesize_MB=32768` in `OPENIMAGEIO_OPTIONS`
  unless the test sets it, so corrupt-file errors are the same on every
  machine.
- `TESTSUITE_CLEANUP_ON_SUCCESS=1` deletes outputs of passing tests.
- For failures only seen in CI, download the CI artifact and inspect the
  real output before touching refs (TESTSUITE-README.md).
