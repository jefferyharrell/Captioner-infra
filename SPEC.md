# 1. Captioner – Project Spec (LLM-Optimized, Tabular)

**Spec Version:** 1.0.3  
**Last Updated:** 2025-04-19

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
    - **Filesystem**: Reads photos from a local directory.
    - **Amazon S3**: Uses S3 HTTP API for photo storage/retrieval.
- The abstraction allows for easy switching or extension to new storage providers in the future.
- All implementations must conform to the same interface and support the required operations efficiently.

## 5. Backend API Endpoints

| Method | Path                        | Purpose                  | Request Example                  | Response Example                |
|--------|-----------------------------|--------------------------|----------------------------------|---------------------------------|
| POST   | /login                      | Authenticate user        | `{ "password": "..." }`         | `{ "success": true }`           |
| GET    | /photos?limit=100&offset=0  | List photo IDs           |                                  | `{ "photo_ids": [1,2,3] }`      |
| GET    | /photos/{id}                | Get photo metadata       |                                  | `{ "id": 1, ... }`              |
| PATCH  | /photos/{id}/caption        | Update photo caption     | `{ "caption": "Nice" }`         | `{ "id": 1, "caption": "Nice"}` |
| GET    | /photos/{id}/image          | Get image file           |                                  | (binary image)                  |
| POST   | /rescan                     | Rescan image folder      |                                  | `{ "status": "ok" }`            |
| GET    | /photos/shuffled?limit=100  | Get shuffled photo IDs   |                                  | `{ "photo_ids": [3,1,2] }`      |

**Parameter Constraints:**
- `limit` (integer): Default 100, min 1, max 1000
- `offset` (integer): Default 0, min 0

## 6. Data Model

| Field     | Type     | Constraints                 | Description                   |
|-----------|----------|----------------------------|-------------------------------|
| id        | integer  | PK, auto-increment         | Unique photo ID               |
| hash      | string   | SHA-256, unique, indexed   | Used for deduplication        |
| filename  | string   |                            | Original filename             |
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
- No upload/delete UI; images identified by SHA-256 hash.

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
| DROPBOX_TOKEN           | Access token for Dropbox API                   | string  | No       |                        |
| S3_BUCKET               | AWS S3 bucket name                             | string  | No       |                        |
| AWS_ACCESS_KEY_ID       | AWS access key ID for S3 authentication        | string  | No       |                        |
| AWS_SECRET_ACCESS_KEY   | AWS secret access key for S3 authentication    | string  | No       |                        |

- All configuration via environment variables.
- Local: use `.env` (not in git); reference with `docker-compose` (`env_file:`) or `docker run --env-file`.
- Never hardcode secrets in Dockerfiles; inject at runtime.
- Production: use Docker secrets or cloud secret manager.
- Env vars loaded at container start, not build time.

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
| Hash     | SHA-256 hash of image file, used as unique ID       |
| LRU      | Least Recently Used cache for thumbnails            |

## 15. Example API Usage

### Get photo IDs

Request: `GET /photos?limit=2&offset=0`

Response: `{ "photo_ids": [1, 2] }`

### Get photo metadata

Request: `GET /photos/1`

Response: `{ "id": 1, "hash": "...", "filename": "foo.jpg", "caption": "A dog" }`

### Update caption

Request: `PATCH /photos/1/caption` with `{ "caption": "A better caption" }`

Response: `{ "id": 1, "caption": "A better caption" }`

### Error response

Response: `{ "detail": "Photo not found" }` (404)
