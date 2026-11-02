# Agent Instructions

> `CLAUDE.md` is a symlink to this file. Edit `AGENTS.md`.

An optional, gitignored `AGENTS.local.md` holds per-developer overrides layered on top of this file. It is simply absent for anyone who has not created one.

@AGENTS.local.md

## Repository Overview

`aidb` is a live-attach debugger for multi-agent apps (Python >= 3.11, no runtime dependencies). See [`README.md`](README.md).

- `aidb/` — the debugger: `DebugSession` gate in the app's runtime, a localhost control server, and the `aidb` CLI (`python -m aidb <command> --port <n>`).
- `samples/refund_desk/` — mocked multi-agent sample app the debugger attaches to.
- `tests/` — pytest suite. `docs/product/` — product docs.

## Preferred Commands

```bash
python -m pip install -e ".[dev]"   # pytest, ruff, mypy
python -m ruff check .              # lint
python -m mypy aidb                 # type check
python -m pytest -q                 # tests (one-shot; testpaths = tests)
python -m pytest tests/test_edit_resume.py -q   # scope to one file
```

The only CI is [`.github/workflows/ocr-review.yml`](.github/workflows/ocr-review.yml) (OpenCodeReview PR review).

## Hard Rules (Non-Negotiable)

If a task appears to require breaking one of these, stop and ask the human first.

### Never push to `main`

Do not `git push` to `main` (including `--force`). All changes land through a reviewed pull request. Force-pushing a feature branch is fine; prefer `--force-with-lease`.

### Never leave a process you started running

Every process you start is yours to stop: sample runs (`python -m samples.refund_desk --debug-port 8765 --pace 2`), `python -m aidb attach` sessions, anything holding a control port.

- Prefer one-shot foreground commands. Never run pytest in a watch loop; give long commands an explicit timeout.
- Before backgrounding anything, know how you will stop it (PID, `python -m aidb end --port <n>`).
- Check first (`lsof -ti :8765`) and reuse what is already up instead of starting a second copy.
- Before reporting done, verify nothing you started is alive (`lsof -ti :8765`, `pgrep -fl 'samples.refund_desk|aidb'`) and kill stragglers.
- Exception: the user explicitly asks you to leave something running. Then say what, on which port, and how to stop it.

## Code Style

- **No unnecessary comments.** Names and types carry the meaning. Comment only a non-obvious _why_ (workaround, invariant, trade-off), in one terse line.
- **Never commit secrets**, including real control tokens (`AIDB_CONTROL_TOKEN`) or credentials.
- **Reuse before adding.** After writing a helper, `rg` for existing code doing the same and consolidate.
- **Unique test fixtures.** Never share hard-coded ids (order ids, session tokens, ports) across tests; generate a unique value per test. Bind test servers to port 0 rather than fixed ports.

## Testing Policy

- **Tests before code.** Write the failing pytest test first, including for new branches or edge cases in existing code. A test shaped to fit finished code only restates it.
- **Prefer end-to-end tests.** Drive a real `DebugSession` through the control server or CLI (see `tests/test_attach_halt.py`, `tests/test_edit_resume.py`) over mocking internals.
- For isolated tests, list the ways the unit can fail first, test each, then implement. Reserve them for logic E2E cannot reach (parsing, validation, state transitions).
- No trivial tests: constants, defaults, `__repr__`, or mocks returning what they were told to. Delete them when found. Coverage percentage is not a goal.

## Commit and PR Guidance

- Keep commits atomic, focused, and descriptive.
- Before every push, run `python -m ruff check .`, `python -m mypy aidb`, and `python -m pytest -q`; never push with failures.
- **Never add agent co-author trailers** (`Co-Authored-By: Claude`, etc.) to commits.
- **Never attribute a PR to an agent**: no "Generated with …" lines or badges in titles, bodies, or comments.
- **Never commit screenshots or PR media.** Save them outside the git tree and share the path so the human attaches them.
- Include docs updates (`README.md`, `docs/`) with behavior changes.

## AI Assistant Guidance

- **Change only what the task requires, and finish it.** Match neighboring patterns; leave unrelated cleanups for their own PR.
- Validate assumptions against this repo's `pyproject.toml` and `README.md`, not other repositories' conventions.

### Subagent Model Selection

Right-size the model instead of inheriting the parent's. When in doubt, pick the lower tier.

- **Haiku** — mechanical, narrow work: file lookups, grep/find, running a command and reporting output, rename/format/lint-fix edits.
- **Sonnet** — bounded synthesis: multi-file edits with settled design, writing tests for an understood module, reviewing a bounded diff, drafting a PR description.
- **Opus** — real thinking only: architecture, non-obvious debugging, protocol/API design, cross-module refactors, security review.
