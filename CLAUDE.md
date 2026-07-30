# coinshop

Early-stage Python project. No established architecture yet — check with the
user before introducing frameworks, dependencies, or structural conventions
that aren't already reflected in the code.

## Conventions

- Python >= 3.11, source under `src/coinshop/`, tests under `tests/`.
- Dependency management is `uv`-only: `uv add`/`uv add --dev` to change
  dependencies, `uv sync` to install. Don't hand-edit `uv.lock`; don't use
  `pip install` directly. Commit `uv.lock`.
- Ruff runs with `select = ["ALL"]` and mypy runs with `strict = true`
  (see `pyproject.toml`) — this is deliberate, not accidental strictness.
  Fix violations rather than widening the ignore list unless a rule is
  genuinely wrong for this project.
- Before proposing a change is done, run all four:
  `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy .`,
  `uv run pytest`.
- This is a solo public repo — no CI-only conventions to preserve, but keep
  `main` green: don't land changes that fail any of the above.

## Security

- Never commit secrets, tokens, or credentials. `.env*`, `*.pem`, `*.key` are
  gitignored — keep it that way.
- Report vulnerabilities per `SECURITY.md`, not as public issues.
