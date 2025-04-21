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
- The backend MUST store images ("blobs") and associated metadata fields (currently "description").
- The storage mechanism for descriptions is backend-dependent (see Section 3).

## 2. MVP Scope
- Only the Dropbox backend is REQUIRED for MVP.
- S3 and filesystem backends are post-MVP and MAY never be implemented.
- All references to S3/filesystem in this document are for extensibility/future-proofing only.

## 3. Storage Abstraction
- The backend MUST expose a storage abstraction with the following interface:
  - `list_photos() -> list[int]`: MUST list all available photo object IDs
  - `get_photo(photo_id: int) -> Photo`: MUST retrieve a specific photo (by ID or hash), including description
  - `set_metadata(photo_id: int, metadata: dict) -> Photo`: MUST update a photo's metadata fields. The `metadata` argument MUST be a JSON object. For MVP, this object will only contain `{"description": "..."}`, but the interface is designed for future extensibility (e.g., additional fields such as tags, author, etc.). The returned `Photo` object MUST reflect the updated metadata.
- **Backend-specific behavior:**
  - **Dropbox backend:**
    - Blobs MUST be stored in Dropbox
    - Descriptions MUST be stored as XMP metadata in the dc:description field of the image file (no DB required)
  - **S3 backend (future):**
    - Blobs MUST be stored in S3
    - Descriptions MUST be stored as XMP metadata in the dc:description field of the image file (no DB required)
  - **Filesystem backend (future):**
    - Blobs MUST be stored in local filesystem
    - Descriptions MUST be stored as XMP metadata in the dc:description field of the image file (no DB required)
- The API and abstraction MUST be unified; implementation MAY vary by backend.

## 4. API Endpoints
| Method | Path                        | Purpose                  | Request Example                  | Response Example                |
|--------|-----------------------------|--------------------------|----------------------------------|---------------------------------|
| POST   | /login                      | Authenticate user        | `{ "password": "hunter2" }`         | `{ "access_token": "<jwt>", "token_type": "bearer" }`           |
| GET    | /photos?limit=100&offset=0  | List photo IDs           |                                  | `{ "photo_ids": [1,2,3] }`      |
| GET    | /photos/{id}                | Get photo data and all metadata fields |                                  | `{ "id": 1, "object_key": "photos/foo.jpg", "description": "A dog" }`              |
| PATCH  | /photos/{id}/metadata    | Update photo metadata (currently only `description`) | `{ "description": "Nice" }`         | `{ "id": 1, "description": "Nice"}` |
| GET    | /photos/{id}/image          | Get image file           |                                  | (binary image)                  |
| POST   | /rescan                     | Discover and sync photos from storage |                                  | `{ "status": "ok" }`            |

- All protected endpoints MUST require `Authorization: Bearer <token>` in the header.
- The `/photos/{id}` and `/photos/{id}/metadata` endpoints MUST use XMP metadata in the dc:description field of the image file for all backends.

## 5. Data Model
| Field      | Type     | Description                                           |
|------------|----------|-------------------------------------------------------|
| id         | integer  | Unique photo ID (DB PK, auto-increment)               |
| object_key | string   | Storage path or object key (Dropbox path or S3 key)   |
| description    | string   | User-supplied description (stored as XMP metadata in the dc:description field; see Section 3)     |

> **Note:** The GET `/photos/{id}` endpoint returns both photo data and all metadata fields as a single JSON object. Additional metadata fields MAY be supported in the future.

## 6. Configuration (Environment Variables)
| Variable                | Purpose                        | Type    | Required | Example Value           |
|-------------------------|--------------------------------|---------|----------|------------------------|
| PASSWORD                | Backend login password         | string  | Yes      | hunter2                |
| STORAGE_BACKEND         | Photo storage backend          | string  | No       | dropbox                |
| DROPBOX_APP_KEY         | Dropbox API app key            | string  | Yes*     | your_app_key           |
| DROPBOX_APP_SECRET      | Dropbox API app secret         | string  | Yes*     | your_app_secret        |
| DROPBOX_REFRESH_TOKEN   | Dropbox OAuth 2.0 refresh token| string  | Yes*     | your_refresh_token     |
| JWT_SECRET_KEY          | JWT signing key                | string  | Yes      | (long random string)   |
| DATABASE_URL            | DB connection string (future/optional use; not required for XMP-only backends) | string  | No       | sqlite:///photos.db    |

*Required for Dropbox backend only.

## 7. Error Codes
| Status | Meaning         | Example Scenario                                |
|--------|----------------|-------------------------------------------------|
| 400    | Bad Request    | Malformed request, invalid parameters           |
| 401    | Unauthorized   | Invalid or missing authentication               |
| 403    | Forbidden      | Authenticated but not allowed                   |
| 404    | Not Found      | Resource does not exist (e.g., photo ID)        |
| 409    | Conflict       | Duplicate resource (e.g., same photo hash)      |
| 422    | Unprocessable  | Semantically invalid (e.g., bad description)        |
| 500    | Server Error   | Unexpected backend/server error                 |

## 8. Development Practices
- Test-driven development (TDD) MUST be followed for all new features.
- All code MUST pass type checking (Pyright) and linting (Ruff) before merging.
- All secrets/configuration MUST be provided via environment variables (never hardcoded).
- CI/CD MUST enforce tests, linting, and type checks.
- Code MUST be concise, maintainable, and PEP 8/257 compliant.

## 9. Roadmap / Extensibility
- S3 and filesystem backends MAY be implemented in the future.
- S3/filesystem backends MUST use XMP metadata in the dc:description field for descriptions/metadata.
- The API/abstraction MUST remain unified regardless of backend.

## 10. Glossary
| Term        | Definition                                                        |
|-------------|-------------------------------------------------------------------|
| Photo       | An image record (blob + metadata)                                 |
| Description     | User-supplied text describing a photo (stored as XMP metadata in the dc:description field; see Section 3 for details) |
| Object Key  | Storage path (Dropbox) or S3 object key                           |
| LRU         | Least Recently Used cache for thumbnails                          |

## 11. Example Usage
### Get photo metadata
Request: `GET /photos/1`
Response: `{ "id": 1, "object_key": "photos/foo.jpg", "description": "A dog" }`

### Update description
Request: `PATCH /photos/1/description` with `{ "description": "A better description" }`
Response: `{ "id": 1, "description": "A better description" }`

### Error response
Response: `{ "detail": "Photo not found" }` (404)

---

## AI Guidance
- When generating or reviewing code, you MUST check which backend is in use and apply the correct metadata storage mechanism.
- For all backends (Dropbox, S3, filesystem), all description CRUD MUST use XMP metadata in the dc:description field of the image file.
- You MUST NOT assume S3/filesystem backends exist unless explicitly enabled.
- All endpoint behaviors, error codes, and data model fields MUST match this spec.
