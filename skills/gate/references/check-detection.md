# Check Detection Heuristics

Look for these signals in the repo root (and nearest ancestor if run from a
subdirectory) and run whatever is found. Skip anything not found — do not
report a missing tool as a failure.

## Node / JavaScript / TypeScript

If `package.json` exists, read its `scripts` block:
- `test` script present → run `npm test` (or `yarn test` / `pnpm test` if
  a `yarn.lock` / `pnpm-lock.yaml` is present instead of `package-lock.json`)
- `lint` script present → run the equivalent lint command
- `typecheck` or `tsc` script present → run it; otherwise if `tsconfig.json`
  exists, run `npx tsc --noEmit`

## Python

- `pyproject.toml` or `pytest.ini` or a `tests/` directory with `test_*.py`
  files → run `pytest`
- `pyproject.toml` with a `[tool.ruff]` section, or a `.ruff.toml` /
  `ruff.toml` file → run `ruff check .`
- `mypy.ini` or `pyproject.toml` with `[tool.mypy]` → run `mypy .`

## Go

- `go.mod` present → run `go test ./...`
- Run `go vet ./...` unconditionally when `go.mod` is present

## Rust

- `Cargo.toml` present → run `cargo test`
- Run `cargo clippy` if available

## Makefile fallback

If a `Makefile` exists, prefer its `test` / `lint` / `typecheck` targets
over the ecosystem-specific commands above when both exist (the Makefile
target is assumed to wrap the correct invocation for the project).

## Reporting

For each detected and run check, capture: command run, exit code, and
relevant output (failing test names, lint violations with file:line).
A check with no detectable command for its category is omitted from the
report entirely — never listed as skipped or failed.
