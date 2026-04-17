# Read Chinese With Sino-Vietnamese - API Surface (MVP)

This document defines a high-level, implementation-agnostic API surface based on:
- `APP_REQUIREMENTS_CONTRACT.md` (v0.4)
- `HIGH_LEVEL_DESIGN.md` (MVP high-level components)

It describes component interfaces, not internal algorithms.

## 1) API Style and Conventions

- Protocol: HTTPS + JSON for metadata APIs.
- Auth: bearer session token (or secure session cookie) after login.
- Scope: all resources are user-scoped.
- Time format: ISO-8601 UTC timestamps.
- Encoding: full Unicode support (Chinese, pinyin tone marks, Vietnamese diacritics).
- Error shape: consistent envelope with user-actionable message.
- Credential handling:
  - `password` typed as `string` means JSON data type only.
  - credentials are sent over TLS (HTTPS) in transit.
  - server stores only secure password hashes (never plaintext).

Example error envelope:

```json
{
  "error": {
    "code": "LOOKUP_SOURCE_TIMEOUT",
    "message": "Lookup source timed out. Please try again.",
    "retryable": true
  }
}
```

## 2) Component API Map

## 2.1 Client UI -> Application API (public app surface)

### Authentication

#### `POST /api/v1/auth/login`
- Purpose: authenticate with username/password and establish session.
- Security note: request includes raw password input for verification, but transport MUST be HTTPS/TLS and backend MUST hash passwords at rest.
- Request:
```json
{
  "username": "string",
  "password": "string"
}
```
- Response `200`:
```json
{
  "user": {
    "id": "user_123",
    "username": "learner_a"
  },
  "session": {
    "token": "opaque-or-jwt",
    "expiresAt": "2026-04-18T10:00:00Z"
  }
}
```
- Errors: `401` invalid credentials (non-leaking message), `429` rate limit.

#### `POST /api/v1/auth/logout`
- Purpose: terminate current session.
- Response `204`.

#### `GET /api/v1/auth/session`
- Purpose: check current auth/session state.
- Response `200` with active session summary, or `401` if expired.

---

### Document Library and Reading

#### `POST /api/v1/documents:import`
- Purpose: import new document (`EPUB`, `TXT`, `HTML`).
- Content type: `multipart/form-data`.
- Fields:
  - `file`: binary file
  - `title` (optional): user-friendly document name
- Response `201`:
```json
{
  "document": {
    "id": "doc_abc",
    "title": "My Chinese Reader",
    "sourceFormat": "EPUB",
    "createdAt": "2026-04-17T09:00:00Z"
  }
}
```
- Errors:
  - `400` unsupported format / empty file
  - `422` parse failure

#### `GET /api/v1/documents`
- Purpose: list existing documents for "Open existing document" flow.
- Response `200`:
```json
{
  "items": [
    {
      "id": "doc_abc",
      "title": "My Chinese Reader",
      "sourceFormat": "EPUB",
      "updatedAt": "2026-04-17T09:10:00Z"
    }
  ]
}
```

#### `GET /api/v1/documents/{documentId}`
- Purpose: fetch document metadata.
- Response `200` with document summary.
- Errors: `404` not found (or not owned by user).

#### `GET /api/v1/documents/{documentId}/content`
- Purpose: fetch renderable reading content.
- Response `200`:
```json
{
  "documentId": "doc_abc",
  "title": "My Chinese Reader",
  "content": {
    "type": "normalizedTextBlocks",
    "blocks": [
      {
        "id": "b1",
        "text": "中文内容..."
      }
    ]
  }
}
```
- Notes:
  - exact rendering payload is implementation-defined.
  - must preserve Unicode fidelity.

---

### Lookup

#### `POST /api/v1/lookups`
- Purpose: execute explicit lookup when user clicks `Look up`.
- Document coupling: lookup is document-independent in MVP. It operates on selected text; `documentId` is optional telemetry/context only if needed by implementation.
- Request:
```json
{
  "selection": {
    "text": "中文词"
  }
}
```
- Optional request extensions (not required by contract):
  - `selection.startOffset`, `selection.endOffset`: useful if tokenizer/disambiguation needs exact position in passage.
  - `context.leftText`, `context.rightText`: useful when phrase meaning depends on surrounding text.
  - `documentId`: useful for analytics/debugging, not required for lexical correctness.
- Response `200`:
```json
{
  "result": {
    "text": "中文词",
    "chineseDefinition": "中文释义",
    "pinyin": "Zhōng wén cí",
    "hanViet": "Trung văn từ",
    "vietnameseDefinition": "Từ tiếng Trung"
  },
  "meta": {
    "cacheHit": true
  }
}
```
- Partial response allowed (contract): missing fields must be explicit:
```json
{
  "result": {
    "text": "中文词",
    "chineseDefinition": "Not available",
    "pinyin": "Zhōng wén cí",
    "hanViet": "Trung văn từ",
    "vietnameseDefinition": "Not available"
  }
}
```
- Errors: `503` temporary lookup source failure, `400` invalid selection.

UI timing contract note:
- Double-click only creates selection and shows `Look up`.
- Client applies yellow highlight at same click that calls this endpoint.

---

### New Word List

New Word List is user-scoped and shared across all documents.

#### `POST /api/v1/new-words`
- Purpose: add current lookup result to user list.
- Request:
```json
{
  "text": "中文词/短语",
  "chineseDefinition": "中文释义",
  "pinyin": "Pīn yīn",
  "hanViet": "Hán Việt",
  "vietnameseDefinition": "Nghĩa tiếng Việt"
}
```
- Success `201`:
```json
{
  "item": {
    "id": "nw_001",
    "text": "中文词/短语",
    "chineseDefinition": "中文释义",
    "pinyin": "Pīn yīn",
    "hanViet": "Hán Việt",
    "vietnameseDefinition": "Nghĩa tiếng Việt",
    "createdAt": "2026-04-17T09:20:00Z"
  }
}
```
- Duplicate `409` (add-only dedupe policy):
```json
{
  "error": {
    "code": "NEW_WORD_ALREADY_EXISTS",
    "message": "This item is already in your New Word List.",
    "retryable": false
  },
  "existingItemId": "nw_001"
}
```

#### `GET /api/v1/new-words`
- Purpose: list user saved entries.
- Response `200`:
```json
{
  "items": [
    {
      "id": "nw_001",
      "text": "中文词/短语",
      "chineseDefinition": "中文释义",
      "pinyin": "Pīn yīn",
      "hanViet": "Hán Việt",
      "vietnameseDefinition": "Nghĩa tiếng Việt",
      "createdAt": "2026-04-17T09:20:00Z"
    }
  ]
}
```

#### `DELETE /api/v1/new-words/{itemId}`
- Purpose: remove one saved entry.
- Response `204`.
- Errors: `404` not found.

## 2.2 Application API -> Internal Services (orchestrator contracts)

These are logical service interfaces behind the API gateway/orchestrator.

### Authentication Service
- `authenticate(username, password) -> { user, session }`
- `validateSession(token) -> { userId, expiresAt }`
- `invalidateSession(sessionId) -> void`

### Document Service
- `importDocument(userId, file, title?) -> Document`
- `listDocuments(userId) -> Document[]`
- `getDocument(userId, documentId) -> Document`
- `getDocumentContent(userId, documentId) -> RenderableContent`

### Lookup Service
- `lookup(selection, context?, userId?) -> LookupResultWithMeta`
- `normalizeMissingFields(result) -> LookupResult`
- `getCachedLookup(key) -> LookupResult?` (optional)
- `setCachedLookup(key, result, ttl) -> void` (optional)

### New Word List Service
- `addItem(userId, LookupResult) -> NewWordListItem | DuplicateError`
- `listItems(userId, paging?) -> NewWordListItem[]`
- `deleteItem(userId, itemId) -> void`

## 3) Response and Error Code Matrix

- `200 OK`: successful read/lookup.
- `201 Created`: successful creation (import, new word add).
- `204 No Content`: successful delete/logout.
- `400 Bad Request`: invalid input (selection/format).
- `401 Unauthorized`: invalid/expired session.
- `403 Forbidden`: user does not own resource.
- `404 Not Found`: missing resource.
- `409 Conflict`: duplicate New Word List add.
- `422 Unprocessable Entity`: parse/import semantic failure.
- `429 Too Many Requests`: auth or lookup throttling.
- `503 Service Unavailable`: temporary lookup source outage.

## 4) Minimal OpenAPI-Style Skeleton (non-binding)

```yaml
openapi: 3.1.0
info:
  title: Read Chinese With Sino-Vietnamese API
  version: 1.0.0-mvp
paths:
  /api/v1/auth/login:
    post: {}
  /api/v1/auth/logout:
    post: {}
  /api/v1/auth/session:
    get: {}
  /api/v1/documents:import:
    post: {}
  /api/v1/documents:
    get: {}
  /api/v1/documents/{documentId}:
    get: {}
  /api/v1/documents/{documentId}/content:
    get: {}
  /api/v1/lookups:
    post: {}
  /api/v1/new-words:
    get: {}
    post: {}
  /api/v1/new-words/{itemId}:
    delete: {}
```

## 5) Out-of-Scope Endpoints (MVP clarity)

No API surface is defined yet for:
- spaced repetition scheduling
- Anki integration
- audio pronunciation
- collaborative sync
- direct Kindle/AZW import
