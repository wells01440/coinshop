# coinshop

Early-stage Python project. No established architecture yet — check with the
user before introducing frameworks, dependencies, or structural conventions
that aren't already reflected in the code.

## Conventions

- Python >= 3.11, source under `src/coinshop/`, tests under `tests/`.
- Lint/format with `ruff check .`; test with `pytest`. Run both before
  proposing a change is done.
- Keep `pyproject.toml` as the single source of dependency/tooling config.
- This is a solo public repo — no CI-only conventions to preserve, but keep
  `main` green: don't land changes that fail `ruff` or `pytest`.

## Security

- Never commit secrets, tokens, or credentials. `.env*`, `*.pem`, `*.key` are
  gitignored — keep it that way.
- Report vulnerabilities per `SECURITY.md`, not as public issues.
