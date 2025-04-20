# 1. Captioner – Project Spec (LLM-Optimized, Tabular)

**Spec Version:** 1.2.0  
**Last Updated:** 2025-04-20

---

**Changelog 1.2.0 (2025-04-20):**
- Backend now implements JWT-based authentication for login and protected endpoints.
- /login returns a JWT access token on success.
- All protected endpoints require a valid bearer JWT.
- JWT_SECRET_KEY is required in the environment and enforced at startup.
- Full TDD, type, and lint coverage for authentication code.

**Changelog 1.1.0 (2025-04-19):**
- DropboxStorage now uses OAuth 2.0 with refresh token flow for authentication.
- Static DROPBOX_TOKEN is deprecated for runtime use; backend now requires DROPBOX_APP_KEY, DROPBOX_APP_SECRET, and DROPBOX_REFRESH_TOKEN.
- Backend dynamically obtains short-lived access tokens using the refresh token; access tokens are never stored on disk.
- Spec and config updated to reflect new Dropbox authentication model.

**Changelog 1.0.4 (2025-04-19):**
- DropboxStorage supports pagination for all JPEG/PNG images (recursive, unlimited count)
- Filtering for JPEG/PNG is done client-side (Dropbox API does not support extension filtering)
- All DropboxStorage error paths (missing token, API errors, pagination errors) are fully tested and covered
- PhotoStorage ABC methods now explicitly raise NotImplementedError
- Test coverage is >95% and enforced in CI
- Pre-commit hooks (Ruff, Pyright, coverage) are enforced
- Conventional Commits and git commit message newline rule added to project rules


## 2. Purpose
Private web app for viewing and captioning photos. FastAPI backend, Next.js frontend, SQLite DB, local image storage.

## 3. Architecture
- **Frontend:** Next.js 14 (TypeScript), Tailwind v4, shadcn/ui
- **Backend:** FastAPI (Python 3), SQLite, SQLAlchemy ORM
- **Image Storage:** Pluggable storage backend via environment variable, default: Dropbox via HTTP API
- **Supported Formats:** JPEG, PNG, WEBP (others rejected)
- **Testing:** Pytest (backend), Jest/React Testing Library, Playwright (frontend)
- **OS:** macOS, Linux

## 4. Backend Photo Storage Abstraction
- The backend implements a photo storage abstraction layer with two core functions:
    - `list_photos`: List all available photo objects (IDs, metadata, etc.).
    - `get_photo`: Retrieve a specific photo (by ID or hash).
- The storage backend is pluggable. Supported implementations:
    - **Dropbox** (default): Uses Dropbox HTTP API for photo storage/retrieval.
        - Authenticates via OAuth 2.0: requires `DROPBOX_APP_KEY`, `DROPBOX_APP_SECRET`, and `DROPBOX_REFRESH_TOKEN`.
        - Backend dynamically obtains short-lived access tokens using the refresh token and app credentials.
        - Static `DROPBOX_TOKEN` is no longer used at runtime (may be present for legacy/manual testing only).
        - Access tokens are stored in memory only and never written to disk.
    - **Filesystem**: Reads photos from a local directory.
    - **Amazon S3**: Uses S3 HTTP API for photo storage/retrieval.
- The abstraction allows for easy switching or extension to new storage providers in the future.
- All implementations must conform to the same interface and support the required operations efficiently.

## 5. Backend API Endpoints

| Method | Path                        | Purpose                  | Request Example                  | Response Example                |
|--------|-----------------------------|--------------------------|----------------------------------|---------------------------------|
| POST   | /login                      | Authenticate user        | `{ "password": "..." }`         | `{ "access_token": "<jwt>", "token_type": "bearer" }`           |
| GET    | /photos?limit=100&offset=0  | List photo IDs           |                                  | `{ "photo_ids": [1,2,3] }`      |
| GET    | /photos/{id}                | Get photo metadata       |                                  | `{ "id": 1, ... }`              |
| PATCH  | /photos/{id}/caption        | Update photo caption     | `{ "caption": "Nice" }`         | `{ "id": 1, "caption": "Nice"}` |
| GET    | /photos/{id}/image          | Get image file           |                                  | (binary image)                  |
| POST   | /rescan                     | Discover and sync photos from storage |                                  | `{ "status": "ok" }`            |
| GET    | /photos/shuffled?limit=100  | Get shuffled photo IDs   |                                  | `{ "photo_ids": [3,1,2] }`      |

All protected endpoints require `Authorization: Bearer <token>` in the header.

### /rescan – Image Discovery Endpoint

- **Purpose:** Walks the configured photo store (Dropbox, S3, or filesystem) and ensures every image file has a corresponding record in the database. If a photo exists in storage but not in the database, a new record is created. No records are deleted or modified for missing files.
- **When to use:** Call this endpoint after adding images directly to storage (e.g., Dropbox folder, S3 bucket, or local directory) outside of the app.
- **Request:** `POST /rescan`
- **Response:** `{ "status": "ok", "num_new_photos": 3 }`
  (where `num_new_photos` is the number of new records created)
- **Notes:**
  - Only supported image formats (JPEG, PNG, WEBP) are discovered.
  - Does not delete or modify existing database records for missing files.
  - May be a long-running operation for large stores.

**Parameter Constraints:**
- `limit` (integer): Default 100, min 1, max 1000
- `offset` (integer): Default 0, min 0

## 6. Data Model

| Field     | Type     | Constraints                 | Description                   |
|-----------|----------|----------------------------|-------------------------------|
| id         | integer  | PK, auto-increment         | Unique photo ID               |
| object_key| string   | unicode, unique, indexed   | Storage path or S3 object key |
| caption   | text     | nullable                   | User-supplied caption         |

## 7. Error Codes

| Status | Meaning         | Example Scenario                                |
|--------|----------------|-------------------------------------------------|
| 400    | Bad Request    | Malformed request, invalid parameters           |
| 401    | Unauthorized   | Invalid or missing authentication               |
| 403    | Forbidden      | Authenticated but not allowed                   |
| 404    | Not Found      | Resource does not exist (e.g., photo ID)        |
| 409    | Conflict       | Duplicate resource (e.g., same photo hash)      |
| 422    | Unprocessable  | Semantically invalid (e.g., bad caption)        |
| 500    | Server Error   | Unexpected backend/server error                 |

## 8. Frontend Features

- Single-photo random UX; displays one random photo at a time.
- Editable caption field below image; saves edits in real time (debounced).
- No gallery/grid view.
- No rescan button in UI.
- Responsive design; works on all modern browsers.
- Basic accessibility: alt text is caption > filename > blank.
- No upload/delete UI; images are identified by database id and object_key.

## 9. Project Structure

- `/backend/app/` – FastAPI app and models
- `/backend/images/` – Original images
- `/backend/thumbnails/` – Cached thumbnails
- `/backend/photos.db` – SQLite database
- `/backend/tests/` – Pytest tests
- `/frontend/src/` – React source code
- `/frontend/public/` – Static files
- `/frontend/package.json` – Frontend dependencies

## 10. Configuration & Environment Variables

| Variable                | Purpose                        | Type    | Required | Example Value           |
|-------------------------|--------------------------------|---------|----------|------------------------|
| PASSWORD                | Backend login password         | string  | Yes      | hunter2                |
| THUMBNAIL_CACHE_SIZE_MB | LRU thumbnail cache size (MB)  | int     | No       | 128                    |
| FRONTEND_API_KEY        | Shared secret for API          | string  | No       | abc123                 |
| DATABASE_URL            | DB connection string           | string  | No       | sqlite:///photos.db    |
| STORAGE_BACKEND         | Photo storage backend (dropbox|filesystem|s3), default: dropbox | string  | No       | dropbox                |
| DROPBOX_APP_KEY         | Dropbox API app key (OAuth 2.0)                | string  | Yes*     | your_app_key           |
| DROPBOX_APP_SECRET      | Dropbox API app secret (OAuth 2.0)             | string  | Yes*     | your_app_secret        |
| DROPBOX_REFRESH_TOKEN   | Dropbox OAuth 2.0 refresh token                | string  | Yes*     | your_refresh_token     |
| DROPBOX_TOKEN           | (Deprecated) Access token for Dropbox API      | string  | No       |                        |
| S3_BUCKET               | AWS S3 bucket name                             | string  | No       |                        |
| AWS_ACCESS_KEY_ID       | AWS access key ID for S3 authentication        | string  | No       |                        |
| AWS_SECRET_ACCESS_KEY   | AWS secret access key for S3 authentication    | string  | No       |                        |
| JWT_SECRET_KEY          | Secret key for signing/verifying JWT tokens         | string  | Yes      | (long random string)         |

*Required for Dropbox backend only. Not required for filesystem or S3.

- Dropbox backend uses OAuth 2.0: the backend exchanges the refresh token, app key, and app secret for a short-lived access token at runtime. Access tokens are never stored on disk or in the environment file.
- All configuration via environment variables.
- Local: use `.env` (not in git); reference with `docker-compose` (`env_file:`) or `docker run --env-file`.
- Never hardcode secrets in Dockerfiles; inject at runtime.
- Production: use Docker secrets or cloud secret manager.
- Env vars loaded at container start, not build time.

## Authentication Model

- Users authenticate via POST /login with the backend password.
- On success, the backend returns a JWT access token.
- All protected endpoints require the token in the Authorization header as a Bearer token.
- The JWT is signed with JWT_SECRET_KEY, which must be set in the environment.
- Tokens are validated on every protected request.

## 11. Development Practices

- Test-driven development (TDD) strictly followed.
- All new features require corresponding tests before code is written.
- Tests must pass before code is considered complete.
- Static analysis (Pyright) and style (Ruff) enforced via pre-commit hooks and CI.
- Continuous Integration (CI) with GitHub Actions for both frontend and backend.
- Tests ensure DB/filesystem isolation and clean up after themselves.
- Code is concise, maintainable, and written with skepticism.

## 12. Known Gaps / TODOs (Post-MVP)

- Search functionality (by caption, filename, or metadata)
- ML-based auto-captioning or caption suggestions

## 13. Non-Goals / Out of Scope

- No user accounts or authentication beyond simple password
- No image manipulation or editing
- No gallery/grid view
- No upload/delete UI in MVP

## 14. Glossary

| Term     | Definition                                          |
|----------|-----------------------------------------------------|
| Photo    | An image record in the database and filesystem      |
| Caption  | User-supplied text describing a photo               |
| Object Key | Unicode storage path (Dropbox) or S3 object key      |
| LRU      | Least Recently Used cache for thumbnails            |

## 15. Example API Usage

### Get photo IDs

Request: `GET /photos?limit=2&offset=0`

Response: `{ "photo_ids": [1, 2] }`

### Get photo metadata

Request: `GET /photos/1`

Response: `{ "id": 1, "object_key": "photos/foo.jpg", "caption": "A dog" }`

### Update caption

Request: `PATCH /photos/1/caption` with `{ "caption": "A better caption" }`

Response: `{ "id": 1, "caption": "A better caption" }`

### Error response

Response: `{ "detail": "Photo not found" }` (404)
