# Captioner Project Specification (LLM-Optimized)

**Spec Version:** 2.0.0
**Last Updated:** 2025-04-21

**Intended Audience:** Large Language Model (LLM) AIs, code generation agents, and human developers. Primary purpose: unambiguous, machine-readable reference for automated reasoning and implementation.

---

## RFC 2119 Terminology
This specification uses the key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt). These terms are to be interpreted as follows:
- **MUST**: This requirement is absolute.
- **SHOULD**: There may exist valid reasons in particular circumstances to ignore a particular item, but the full implications must be understood and carefully weighed before choosing a different course.
- **MAY**: This item is truly optional.

---

## 1. Purpose
- Captioner is a web application for viewing and captioning photos.
- The backend MUST store images ("blobs") and associated metadata ("captions").
- The storage mechanism for captions is backend-dependent (see Section 3) and MUST be implemented according to the backend in use.

## 2. MVP Scope
- Only the Dropbox backend is REQUIRED for MVP.
- S3 and filesystem backends are post-MVP and MAY never be implemented.
- All references to S3/filesystem in this document are for extensibility/future-proofing only.

## 3. Storage Abstraction
- The backend MUST expose a storage abstraction with the following interface:
  - `list_photos() -> list[int]`: MUST list all available photo object IDs
  - `get_photo(photo_id: int) -> Photo`: MUST retrieve a specific photo (by ID or hash), including caption
  - `set_caption(photo_id: int, caption: str) -> Photo`: MUST update a photo's caption
- **Backend-specific behavior:**
  - **Dropbox backend:**
    - Blobs MUST be stored in Dropbox
    - Captions MUST be stored in Dropbox file properties (no DB required)
  - **S3 backend (future):**
    - Blobs MUST be stored in S3
    - Captions MUST be stored in SQLite DB or equivalent metadata store
  - **Filesystem backend (future):**
    - Blobs MUST be stored in local filesystem
    - Captions MUST be stored in SQLite DB or equivalent metadata store
- The API and abstraction MUST be unified; implementation MAY vary by backend.

## 4. API Endpoints
| Method | Path                        | Purpose                  | Request Example                  | Response Example                |
|--------|-----------------------------|--------------------------|----------------------------------|---------------------------------|
| POST   | /login                      | Authenticate user        | `{ "password": "hunter2" }`         | `{ "access_token": "<jwt>", "token_type": "bearer" }`           |
| GET    | /photos?limit=100&offset=0  | List photo IDs           |                                  | `{ "photo_ids": [1,2,3] }`      |
| GET    | /photos/{id}                | Get photo metadata       |                                  | `{ "id": 1, ... }`              |
| PATCH  | /photos/{id}/caption        | Update photo caption     | `{ "caption": "Nice" }`         | `{ "id": 1, "caption": "Nice"}` |
| GET    | /photos/{id}/image          | Get image file           |                                  | (binary image)                  |
| POST   | /rescan                     | Discover and sync photos from storage |                                  | `{ "status": "ok" }`            |

- All protected endpoints MUST require `Authorization: Bearer <token>` in the header.
- The `/photos/{id}` and `/photos/{id}/caption` endpoints MUST use the backend's metadata mechanism (Dropbox file properties for Dropbox, DB for S3/filesystem).

## 5. Data Model
| Field      | Type     | Description                                           |
|------------|----------|-------------------------------------------------------|
| id         | integer  | Unique photo ID (DB PK, auto-increment)               |
| object_key | string   | Storage path or object key (Dropbox path or S3 key)   |
| caption    | string   | User-supplied caption (see Section 3 for storage)     |

## 6. Configuration (Environment Variables)
| Variable                | Purpose                        | Type    | Required | Example Value           |
|-------------------------|--------------------------------|---------|----------|------------------------|
| PASSWORD                | Backend login password         | string  | Yes      | hunter2                |
| STORAGE_BACKEND         | Photo storage backend          | string  | No       | dropbox                |
| DROPBOX_APP_KEY         | Dropbox API app key            | string  | Yes*     | your_app_key           |
| DROPBOX_APP_SECRET      | Dropbox API app secret         | string  | Yes*     | your_app_secret        |
| DROPBOX_REFRESH_TOKEN   | Dropbox OAuth 2.0 refresh token| string  | Yes*     | your_refresh_token     |
| JWT_SECRET_KEY          | JWT signing key                | string  | Yes      | (long random string)   |
| DATABASE_URL            | DB connection string           | string  | No       | sqlite:///photos.db    |

*Required for Dropbox backend only.

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

## 8. Development Practices
- Test-driven development (TDD) MUST be followed for all new features.
- All code MUST pass type checking (Pyright) and linting (Ruff) before merging.
- All secrets/configuration MUST be provided via environment variables (never hardcoded).
- CI/CD MUST enforce tests, linting, and type checks.
- Code MUST be concise, maintainable, and PEP 8/257 compliant.

## 9. Roadmap / Extensibility
- S3 and filesystem backends MAY be implemented in the future.
- S3/filesystem backends MUST use a DB for captions/metadata.
- The API/abstraction MUST remain unified regardless of backend.

## 10. Glossary
| Term        | Definition                                                        |
|-------------|-------------------------------------------------------------------|
| Photo       | An image record (blob + metadata)                                 |
| Caption     | User-supplied text describing a photo (see Section 3 for storage) |
| Object Key  | Storage path (Dropbox) or S3 object key                           |
| LRU         | Least Recently Used cache for thumbnails                          |

## 11. Example Usage
### Get photo metadata
Request: `GET /photos/1`
Response: `{ "id": 1, "object_key": "photos/foo.jpg", "caption": "A dog" }`

### Update caption
Request: `PATCH /photos/1/caption` with `{ "caption": "A better caption" }`
Response: `{ "id": 1, "caption": "A better caption" }`

### Error response
Response: `{ "detail": "Photo not found" }` (404)

---

## AI Guidance
- When generating or reviewing code, you MUST check which backend is in use and apply the correct metadata storage mechanism.
- For Dropbox, all caption CRUD MUST use Dropbox file properties API.
- For S3/filesystem, all caption CRUD MUST use the DB.
- You MUST NOT assume S3/filesystem backends exist unless explicitly enabled.
- All endpoint behaviors, error codes, and data model fields MUST match this spec.
