# Repository Guidelines

## Project Structure & Module Organization
This repository is a MkDocs documentation project.
- `docs/`: all published Markdown content (for example `docs/pto-ir-manual.md`).
- `mkdocs.yml`: site metadata and navigation; every new page should be added here.
- `requirements.txt`: pinned docs toolchain (`mkdocs`, `mkdocs-material`).
- `.github/workflows/deploy-docs.yml`: CI build and GitHub Pages deploy pipeline.
Generated and local-only paths are ignored: `site/`, `.venv/`, and Python cache files.

## Build, Test, and Development Commands
Use a virtual environment for local work:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
Key commands:
- `mkdocs serve`: run local preview at `http://127.0.0.1:8000`.
- `mkdocs build --strict`: production build with strict validation (same quality gate used in CI).
- `mkdocs build`: optional non-strict local build when iterating quickly.

## Coding Style & Naming Conventions
- Write docs in Markdown with clear ATX headings (`#`, `##`, `###`).
- Keep filenames lowercase kebab-case (for example `bin-intrinsic.md`), consistent with existing docs.
- Use relative links inside `docs/` and verify they resolve in local preview.
- Keep `mkdocs.yml` indentation/style consistent with current YAML formatting.
- Prefer concise, technical wording; include command or path examples when useful.

## Testing Guidelines
There is no unit-test suite; documentation validation is the test surface.
- Run `mkdocs build --strict` before opening a PR.
- Confirm new/renamed pages are present in `mkdocs.yml` nav.
- Manually click key links in `mkdocs serve` to catch broken anchors and paths.

## Commit & Pull Request Guidelines
Recent history follows conventional prefixes such as `docs:` and `chore:`. Continue that pattern.
- Commit format: `<type>: <imperative summary>` (example: `docs: add PTO IR memory model section`).
- Keep commits focused to one logical change.
- PRs should include: purpose, files/pages changed, and preview notes or screenshots for navigation/UI-impacting changes.
- Link related issues/tasks when available and ensure docs CI passes before merge.
- 完成代码修改后，需要找我确认是否帮我完成`commit & push & 发布`。