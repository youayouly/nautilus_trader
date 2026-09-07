# Windows Build Notes — nautilus_trader (Rust + MSVC)

Notes from building this repo on Windows (Chinese locale) while working on
`feat/cache-top-of-book` (see #4883). `make` is not available under Git Bash
here, so the `build-debug` target's steps were run manually from `python/`:

```bash
uv sync --all-groups --all-extras --no-install-package nautilus-trader
NAUTILUS_STUB_PROFILE=nextest CARGO_TARGET_DIR=<target> uv run --no-sync python generate_stubs.py
CARGO_TARGET_DIR=<target> uv run --no-sync maturin develop --profile nextest
```

## Gotchas and fixes

1. **`UnicodeDecodeError` in `generate_docstrings.py`** (called by `generate_stubs.py`).
   It reads `.rs` files with the OS default encoding; on Chinese Windows that's
   GBK, which chokes on non-ASCII bytes in source comments/docstrings.
   **Fix:** set `PYTHONUTF8=1` before running.

2. **`maturin develop` picks the wrong Python interpreter.** It can pick up a
   stray Python (e.g. 3.8) from system `PATH` instead of the project venv's
   Python. Since `#[cfg(Py_3_10)]`-gated code in the vendored
   `patches/pyo3-stub-gen` crate (e.g. `PyEncodingWarning`) gets compiled out
   under Python < 3.10, this causes `error[E0425]: cannot find type
   PyEncodingWarning` and a cascade of "could not compile" failures across
   ~15 unrelated adapter crates (tardis, binance, okx, bybit, kraken, etc.)
   — looks like a huge breakage but is unrelated to any actual code diff.
   **Fix:** explicitly set `PYO3_PYTHON` to the venv's `python.exe` path
   before invoking `maturin develop` / `cargo check --features python`.

3. **`rustc.exe` crash (`STATUS_STACK_BUFFER_OVERRUN`, exit `0xc0000409`)**
   while compiling the final `crates/pyo3` cdylib (the step that links
   everything into `nautilus_pyo3`) — looks like a compiler stack overflow
   but isn't. Root cause turned out to be **a Windows parallel-build race
   condition**: with default parallelism (16 jobs on a 16-core machine),
   rustc/cargo intermittently produced bogus errors on crates that had
   *already compiled successfully* moments earlier (`can't find crate for
   nautilus_core`, `error[E0786]: found invalid metadata files for crate
   serde`, `only metadata stub found for dylib dependency std`) — classic
   symptom of AV/filesystem lag racing with cargo's parallel metadata reads
   on Windows.
   **Fix that worked:** `CARGO_BUILD_JOBS=4` (down from the default 16).
   With that alone, both `generate_stubs.py` and
   `maturin develop --profile nextest` completed cleanly on the very next
   attempt (`Finished nextest profile [unoptimized] target(s) in 6m 40s`,
   wheel built, installed editable as `nautilus-trader-2.0.0rc4`).
   If a build fails with cascading "can't find crate X" / "invalid metadata"
   errors for crates that appeared to compile fine earlier in the same log,
   suspect this race first — wipe `target/nextest` and retry with a lower
   `CARGO_BUILD_JOBS` rather than assuming a real code/dependency problem.

4. **Compiled extension module name.** The actual compiled extension is
   `nautilus_trader._libnautilus` (a `.pyd` directly under
   `python/nautilus_trader/`), **not** `nautilus_trader.core.nautilus_pyo3`.
   Verify a successful build with `import nautilus_trader._libnautilus`.

## Summary

Always set `CARGO_BUILD_JOBS=4` (or similar reduced value) for Rust builds
of this repo on Windows to avoid the parallel-build race — this matters more
than the `PYO3_PYTHON` / `PYTHONUTF8` fixes for getting a clean build in the
first place.
