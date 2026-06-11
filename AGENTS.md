# Repository Guidelines

## Project Structure & Module Organization

Core Python code lives in `app/`. Keep API routes in `app/controllers/`, request and response models in `app/models/`, configuration handling in `app/config/`, reusable helpers in `app/utils/`, and video-generation logic in `app/services/`. The main pipeline is `app/services/task.py`.

User entry points are:

- `main.py`: FastAPI server
- `webui/Main.py`: Streamlit interface
- `cli.py`: command-line workflow

Tests are under `test/services/`; fixtures belong in `test/resources/`. Fonts, music, and static files live in `resource/`. Documentation and deployment notes belong in `docs/`. Generated files under `storage/` are runtime artifacts, not source.

## Build, Test, and Development Commands

Use Python 3.11 and the locked dependency set:

```bash
uv python install 3.11
uv sync --frozen
uv run python main.py
uv run streamlit run webui/Main.py --browser.gatherUsageStats=False
uv run python cli.py --video-subject "Example topic"
uv run python -m unittest discover -s test
```

Run a focused test while iterating, for example:

```bash
uv run python -m unittest test.services.test_task
```

Docker users can run `docker compose up`.

## Coding Style & Naming Conventions

Follow existing Python style: four-space indentation, `snake_case` functions and modules, `PascalCase` classes, and uppercase constants. Add type hints where surrounding code uses them. Keep changes scoped; do not reformat unrelated code or introduce abstractions for one-off behavior. No formatter or linter is currently enforced, so match the edited file.

## Testing Guidelines

Tests use `unittest.TestCase` and `unittest.mock`. Name files `test_<module>.py`, classes `Test<Feature>`, and methods `test_<behavior>`. Mock LLM, TTS, HTTP, Redis, and material-provider calls so unit tests remain deterministic and require no credentials. Add regression tests for bug fixes and run the narrow suite before the full discovery command.

## Commit & Pull Request Guidelines

Use concise Conventional Commit subjects seen in history, such as `feat: add command line generation entry` or `fix(coverr): use download endpoint`. Keep each commit to one logical change.

Pull requests should explain the behavior change, affected entry points, configuration impact, and commands run. Link related issues and include screenshots for WebUI changes. Call out compatibility changes or unverified external-service behavior.

## Security & Configuration

Copy `config.example.toml` to `config.toml` for local settings. Never commit API keys, tokens, passwords, generated videos, or user-specific absolute paths. Preserve TLS verification and existing path-validation controls.
