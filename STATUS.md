# Project Status – Captioner Infrastructure & Backend

**Last updated:** 2025-04-20 06:27 PDT

## Project State

- STATUS.md is now tracked in the infra repo (`Captioner-infra/STATUS.md`).
- All code and infra changes are committed; working tree is clean.
- Python backend uses a venv at `Captioner-backend/.venv` with all dependencies (FastAPI, SQLAlchemy, pytest, etc.).
- FastAPI backend MVP is in progress; initial endpoints (e.g., `/photos`) and infrastructure are implemented, with more API endpoints to come.
- 100% type annotations, Black formatting, and Ruff linting enforced.
- DropboxStorage backend is default, pluggable via env var; all error paths are tested.
- Dropbox backend now uses OAuth 2.0 with refresh token flow for authentication (see SPEC.md v1.1.0).
    - Requires `DROPBOX_APP_KEY`, `DROPBOX_APP_SECRET`, and `DROPBOX_REFRESH_TOKEN` as environment variables.
    - Static `DROPBOX_TOKEN` is deprecated for runtime use; backend dynamically obtains short-lived access tokens using the refresh token.
    - Access tokens are never stored on disk or in the environment file.
- **[In Progress]** Dropbox OAuth refresh token TDD/test coverage is being expanded and improved (branch: `feat/dropbox-oauth-refresh-token`).
- Backend test coverage is 99%+ (well above 90% minimum), all lint/type/tests passing and enforced in CI.
- Pre-commit hooks (Ruff, Pyright, YAML, whitespace, coverage, etc.) are enforced locally and in CI.
- All Dropbox OAuth-only changes, infra, and test cleanup from PR #1 are complete and merged as of 2025-04-20.
- Docker Compose (`Captioner-infra/docker-compose.yml`) injects secrets via environment variables; `.env` is ignored by git and optional for local dev only.
- Project structure, SPEC.md (v1.0.4), and STATUS.md are fully in sync. `/photos` endpoint now returns a list of database row IDs (primary keys), not storage keys or filenames. Photo model uses `object_key` as the canonical storage identifier, matching backend, API, and storage abstraction.
- Configuration is via environment variables only; no secrets in code or version control. See `.env.example` for updated variable list.
- Project rules require Conventional Commits and prohibit newlines in git commit messages.
- Test-driven development (TDD) is strictly followed.

## What Works

- All code, infra, and configuration are committed and up-to-date.
- `/photos` API endpoint is implemented, returns DB row IDs, and is fully TDD-tested (normal, empty, pagination, error cases), with >95% coverage.
- Linting (Ruff), type checking (Pyright), and coverage enforcement are active in pre-commit and CI.
- Docker Compose supports both local dev and production best practices (env var injection, no secrets in code).
- Project structure, SPEC.md, and documentation are consistent and current.
- Storage abstraction (dropbox, filesystem, s3) is implemented and pluggable.

## Next Steps

- Implement `GET /photos/{id}` endpoint for photo metadata.
- Expand and finalize Dropbox OAuth refresh token test coverage (feat/dropbox-oauth-refresh-token).
- Begin frontend integration (once backend endpoints are finalized).
- Expand documentation as new features are added.

## Known Gaps

- Search and ML auto-captioning are out of MVP scope.
- No gallery/grid or upload/delete UI in backend.
- User accounts and authentication not yet implemented.

---

This summary is designed for LLMs, bots, and humans alike. For full details, see SPEC.md, project README, and GitHub.
