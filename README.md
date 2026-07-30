# coinshop

## Status

Early scaffold — no functionality yet.

## Development

Requires [uv](https://docs.astral.sh/uv/) and Python >= 3.11.

```bash
uv sync
```

Run checks locally before pushing:

```bash
uv run ruff check .
uv run ruff format --check .
uv run mypy .
uv run pytest
```

## License

[MIT](LICENSE)
