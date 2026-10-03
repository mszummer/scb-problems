# Ferrite tasks on this branch

This branch adds problems whose deliverable is a **Ferrite application**: a
Rust crate with one dependency (`ferrite-app`) and a `ferrite.json` manifest,
served locally by the Ferrite CLI. They follow the repository's problem
layout, with the differences described here.

The first such problem is [`auction_house`](auction_house/).

## What is the same

- `config.yaml` with the canary header and one entry per checkpoint.
- One `checkpoint_N.md` specification per checkpoint.
- `tests/conftest.py` plus `tests/test_checkpoint_N.py`. Tests take
  `--entrypoint` and `--checkpoint`, are black-box over HTTP, and use the
  `functionality` and `error` markers. Unmarked tests are core.
- `solutions/checkpoint_N/`: a full reference solution per checkpoint.
- Later checkpoints set `include_prior_tests: true`.

## What is different

| Topic | Ferrite problems |
| --- | --- |
| Deliverable | A project directory (`Cargo.toml`, `ferrite.json`, `rust-toolchain.toml`, `src/`), not a Python file. `entry_file` names the manifest |
| Entrypoint | `ferrite dev --dir <project>`. The fixtures append `--port`, `--fresh`, `--clock-ms`, `--instances`, `--no-reload` and `--json` |
| Toolchain | The `ferrite` CLI and the Rust toolchain it builds with must be on `PATH`. Developed against `ferrite` 0.1.12 and Rust 1.89.0 |
| Vendored SDK | A scaffolded Ferrite project carries the SDK under `vendor/`. Reference solutions do not. The fixtures stage a copy of the project in a temporary directory and add the SDK from the installed CLI when it is missing |
| Build output | The staged copy is what gets built, so nothing is written into the submission or this repository |
| Instances | Every service runs with two instances over one set of state, so state kept in process memory fails tests |
| Time | Tests drive the platform clock through `POST /__forge/clock`. That clock reaches the app only as the `published_ms` of each request. `AppContext::now_ms()` stays on wall time under `ferrite dev`, so specifications define the current time as the request's timestamp |
| Admission gate | `ferrite dev` builds and serves source that `ferrite check` would reject. A core test runs `ferrite check` on the project |
| Shutdown | The fixtures start the CLI in its own session and signal the process group. Stopping the CLI alone leaves its app processes running |
| Parallel runs | One project directory serves one app at a time. Do not run a problem's tests with `pytest-xdist` |

## Running a problem's tests

From the repository root, for checkpoint `N` (tests `1` to `N` against
solution `N`):

```bash
uv run --no-project --with pytest --with httpx --with pytest-timeout \
  pytest -p no:cacheprovider \
  auction_house/tests/test_checkpoint_1.py ... auction_house/tests/test_checkpoint_N.py \
  --entrypoint "ferrite dev --dir auction_house/solutions/checkpoint_N" \
  --checkpoint checkpoint_N --timeout=900
```

The first start in a session does a cold `cargo build` (about 20 to 40
seconds here). `config.yaml` sets a long `timeout` for that reason.

## Writing race tests

A defect that only shows when requests race is not caught by racing once.
In `auction_house`, a bug that let two requests with one idempotency key both
take effect appeared in about one racing round in five with two instances.
The race tests therefore repeat independent rounds (`RACE_ROUNDS`). When
adding a race test, run it against a solution that has the defect and check
that it fails.

## Known limits

- **The Harbor converter does not convert these problems.**
  `scripts/scb_to_harbor.py` requires `solutions/checkpoint_1/<entry_file>.py`
  and fails on `auction_house` with
  `entry_file 'ferrite.json' does not exist at .../ferrite.json.py`. Its
  default entry command and its Docker presets are also Python-only. Converting
  a Ferrite problem needs converter support for a non-Python entry, an image
  with Rust and the Ferrite CLI, and an `entrypoint_command` override. None of
  that is done on this branch, so `scripts/check_all_solutions.py` does not
  cover these problems either.
- **Local runtime only.** The behaviour above was observed with the local
  runtime of `ferrite` 0.1.12. It has not been checked against hosted Ferrite,
  where, for example, `published_ms` is documented as possibly unstamped.
- **Lint.** `pyproject.toml` excludes `solutions/` from ruff, and nothing in
  this repository formats or lints the Rust reference solutions.
