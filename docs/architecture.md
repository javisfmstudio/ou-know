# Architecture Design: OU Know

> **Authoring rules (Architect must follow):**
> 1. This answers the **how** the PRD left open. Do not re-open or re-negotiate requirements — if a requirement is wrong or unclear, raise it as a clarification, don't silently redesign around it.
> 2. Every design decision traces back to a requirement (FR-XXX / NFR-XXX). If a component maps to no requirement, justify it as enabling work or cut it.
> 3. Define the contracts between components explicitly — especially anything crossing the Dev-be / Dev-fe boundary. Two agents building against a vague interface will build two incompatible halves.
> 4. State trade-offs and rejected alternatives. A design with no trade-offs is a design that hasn't been thought through.
> 5. Do not assign or schedule work here — that's the Task Breakdown. This document is the "what it looks like," not the "who does what when."
> _(Delete this block before submitting.)_

## Meta

| Field | Value |
|---|---|
| ProjectId | PJ-0002 |
| Author | Architect |
| Date | 2026-09-01 |
| Version | 0.1 |
| Based on PRD version | 0.1 |
| Gate 2 approver | Foan / Mike |
| Status | Draft |

---

## 1. Approach

OU Know is a unified content management platform with four tightly coupled domains: article management, media resources, user accounts, and AI-powered content intelligence. The solution is a **single monolithic backend** with a **modern SPA frontend**, sharing a unified database and session layer. The single most important design decision is **not to split the AI content intelligence layer into a separate service** — its tight coupling with article editing makes independent deployment unnecessary overhead at this project size.

## 2. Architectural Style & Conventions

**Style: Monolith (justified)**

| Aspect | Decision | Why it fits this project's size |
|---|---|---|
| Architectural style | Monolith | 10,000 DAU, 500 concurrent users, single team, four tightly coupled domains. No independent deployability requirement. Monolith eliminates cross-service latency, distributed transaction complexity, and operational overhead. |

**Conventions all code must follow:**
- **Clean-code floor (non-negotiable):** clear naming, single-responsibility functions, no dead or duplicated code, readable structure.
- **Code organization:** Layered architecture — `controllers` (HTTP request/response) → `services` (business logic) → `repositories` (data access) → `models` (entity definitions). No controller-to-repository bypass.
- **Naming conventions:** PascalCase for models, camelCase for API fields, kebab-case for file names. Route paths: `/api/v1/<resource>`.
- **Error handling:** All errors follow a structured `ApiError` shape `{code, message, details}`. No raw exceptions leak to HTTP responses.

## 3. Component Map

| Component | Responsibility | Serves | Detail |
|---|---|---|---|
| Auth | User registration, login, password reset, RBAC enforcement | FR-003, FR-004, FR-005, NFR-002 | §6.1 |
| Article | Article CRUD, status transitions, classification, tagging | FR-001, FR-002 | §6.2 |
| Media | Upload validation, storage, thumbnail generation, article association | FR-003, UF-002 | §6.3 |
| Content Intelligence | SEO metadata generation, AI-assisted writing, tone adjustment | FR-006, FR-007, FR-008 | §6.4 |
| Frontend SPA | Article editor, media library, user management UI | All FRs (presentation layer) | §6.5 |

## 4. System-Wide Interfaces

| From → To | Contract (endpoint / event / data shape) | Notes |
|---|---|---|
| Frontend → Backend | `POST /api/v1/auth/register` `{email, password, role?}` → `{user, token}` | Password must be ≥8 chars |
| Frontend → Backend | `POST /api/v1/auth/login` `{email, password}` → `{user, token}` | 401 on failure, account lockout after 5 failed attempts |
| Frontend → Backend | `POST /api/v1/auth/password-reset/request` `{email}` → `{204}` | Sends reset link via email |
| Frontend → Backend | `POST /api/v1/auth/password-reset/confirm` `{token, newPassword}` → `{204}` | Token valid 1 hour, single-use |
| Frontend → Backend | `GET/POST/PUT/DELETE /api/v1/articles[/:id]` | Article CRUD with role-based access |
| Frontend → Backend | `POST /api/v1/articles/:id/publish` → `{article}` | State transition: draft → published |
| Frontend → Backend | `POST /api/v1/articles/:id/draft` → `{article}` | State transition: published → draft (revert) |
| Frontend → Backend | `POST /api/v1/media/upload` (multipart) → `{media, thumbnail}` | Max 10MB, type-validated |
| Frontend → Backend | `GET /api/v1/media[/:id]` → `{media}` | List / retrieve media items |
| Frontend → Backend | `POST /api/v1/articles/:id/media` `{mediaIds[]}` → `{article}` | Associate media with article |
| Frontend → Backend | `POST /api/v1/content/seo/generate` `{articleId}` → `{title, description, keywords}` | Auto-generate SEO metadata |
| Frontend → Backend | `POST /api/v1/content/ai/suggest` `{articleId, paragraph?, focus?}` → `{suggestions[]}` | AI writing assistance |
| Frontend → Backend | `POST /api/v1/content/tone/adjust` `{articleId, paragraph, tone}` → `{rewritten}` | Tone adjustment: formal/casual |

## 5. Data Model

### Users
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| email | VARCHAR(255) | Unique, indexed |
| password_hash | VARCHAR(255) | bcrypt/argon2 salted hash |
| role | ENUM('admin','editor','author') | RBAC role |
| reset_token | VARCHAR(255) | NULL when no pending reset |
| reset_token_expires | TIMESTAMP | NULL when no pending reset |
| failed_attempts | INT | Reset to 0 on successful login |
| locked_until | TIMESTAMP | NULL when not locked |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Articles
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| title | VARCHAR(500) | Required |
| content | TEXT | Markdown |
| status | ENUM('draft','review','published','archived') | |
| author_id | UUID (FK → Users) | |
| category_id | UUID (FK → Categories) | |
| seo_title | VARCHAR(255) | NULL = auto-generated |
| seo_description | TEXT | NULL = auto-generated |
| seo_keywords | JSON | NULL = auto-generated |
| published_at | TIMESTAMP | NULL when not published |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Media
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| original_filename | VARCHAR(255) | |
| stored_filename | VARCHAR(255) | Unique, system-generated |
| mime_type | VARCHAR(100) | Validated on upload |
| size_bytes | BIGINT | |
| thumbnail_url | VARCHAR(500) | Generated on upload |
| uploaded_by | UUID (FK → Users) | |
| created_at | TIMESTAMP | |

### Article_Media (junction)
| Field | Type | Notes |
|---|---|---|
| article_id | UUID (FK → Articles) | |
| media_id | UUID (FK → Media) | |

### Categories
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(100) | Unique |
| slug | VARCHAR(100) | Unique, URL-friendly |

### Tags
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(100) | Unique |

### Article_Tags (junction)
| Field | Type | Notes |
|---|---|---|
| article_id | UUID (FK → Articles) | |
| tag_id | UUID (FK → Tags) | |

### Password_Reset_Tokens (audit log, referenced by Users.reset_token)
| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| user_id | UUID (FK → Users) | |
| token_hash | VARCHAR(255) | Hashed for storage |
| created_at | TIMESTAMP | Expired after 1 hour |

## 6. Component Detail

### 6.1 Auth (認證與存取控制)
- **Responsibility:** User registration, authentication, password management, RBAC enforcement.
- **Serves:** FR-003, FR-004, FR-005, NFR-002
- **Interfaces:**
  - Inbound: `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/password-reset/request`, `POST /api/v1/auth/password-reset/confirm`
  - Outbound: Email service (for reset links)
- **Data it owns / touches:** Users table, Password_Reset_Tokens table
- **Applicable NFRs:**
  - NFR-002: bcrypt (cost factor 12) for password hashing; TLS for all auth traffic; password_hash stored as salted hash, never plaintext.
  - NFR-001: Login response <100ms p95 (indexed email lookup, no heavy computation).
- **Depends on:** Email service (external), Frontend SPA (login/register pages)
- **Key decisions:**
  - JWT access tokens (15 min TTL) + refresh tokens (7 day TTL, stored as httpOnly cookies).
  - Account lockout: 5 consecutive failed attempts → lock for 15 minutes (FR-003 step 2: "記錄嘗試次數").
  - RBAC middleware: every protected route checks `req.user.role` against route-level role requirements.

### 6.2 Article (文章管理)
- **Responsibility:** Article CRUD, status transitions, classification, tagging.
- **Serves:** FR-001, FR-002
- **Interfaces:**
  - Inbound: `GET/POST/PUT/DELETE /api/v1/articles[/:id]`, `POST /api/v1/articles/:id/publish`, `POST /api/v1/articles/:id/draft`, `POST /api/v1/articles/:id/media`
  - Outbound: Content Intelligence service (auto SEO on publish), Media service (for associated media references)
- **Data it owns / touches:** Articles, Categories, Tags, Article_Media, Article_Tags
- **Applicable NFRs:**
  - NFR-001: Article editor page load <2s at 500 concurrent users — achieved via server-side rendering of article data with pagination, database indexes on `status`, `author_id`, `category_id`.
  - NFR-004: Schema supports 10,000 DAU — articles table indexed for efficient filtering.
- **Depends on:** Auth (RBAC), Content Intelligence (SEO auto-fill), Media (media association)
- **Key decisions:**
  - Status transition table enforced in the service layer (not in DB constraints alone):
    - `draft` → `review` / `published` / `draft` (no-op)
    - `review` → `draft` / `published`
    - `published` → `draft` (revert only; edit requires revert first — FR-002 acceptance criterion)
    - `archived` → `draft` (only admin)
  - `published_at` set on transition to `published`, cleared on revert.
  - Category/Tag lists returned as part of article form payload (single round trip for form rendering — supports NFR-001).

### 6.3 Media (數位資源管理)
- **Responsibility:** Upload validation, storage, thumbnail generation, media retrieval.
- **Serves:** FR-003, UF-002
- **Interfaces:**
  - Inbound: `POST /api/v1/media/upload` (multipart), `GET /api/v1/media[/:id]`
  - Outbound: File system / cloud storage (article uploads), Image processing library (thumbnail generation)
- **Data it owns / touches:** Media, Article_Media
- **Applicable NFRs:**
  - NFR-003: Media storage redundancy (backups) supports uptime SLA.
  - NFR-002: File paths stored, not file contents in DB; access controlled via Auth middleware.
- **Depends on:** Auth (authenticated upload), Article (media association endpoint)
- **Key decisions:**
  - Upload validation: MIME type check (magic bytes) + extension check + size ≤10MB. Reject with 400 if invalid (FR-003).
  - Thumbnails: generated synchronously on upload using sharp/resize to 320px max dimension.
  - Storage: local filesystem under `uploads/` with hashed directory structure (`uploads/ab/cd/...`) to avoid single-directory inode limits. (Cloud storage TBD if scaling beyond single server.)
  - Filename: UUID + original extension to avoid collisions and exposure of original names.

### 6.4 Content Intelligence (內容智慧)
- **Responsibility:** SEO metadata generation, AI-assisted writing, tone adjustment.
- **Serves:** FR-006, FR-007, FR-008
- **Interfaces:**
  - Inbound: `POST /api/v1/content/seo/generate`, `POST /api/v1/content/ai/suggest`, `POST /api/v1/content/tone/adjust`
  - Outbound: External AI service (provider TBD — FR-007/008 depend on this)
- **Data it owns / touches:** Articles (reads content, writes SEO fields), no new tables
- **Applicable NFRs:**
  - NFR-001: AI suggestions response <10s (FR-007 acceptance criterion). Timeout wrapper around external AI call.
  - NFR-004: External AI calls are stateless; rate limiting per user to protect against runaway usage.
- **Depends on:** Auth (user must be logged in), Article (article content retrieval), External AI service
- **Key decisions:**
  - **Do NOT split as a microservice.** Content Intelligence is called during article editing; network hop to a separate service adds latency to an already latency-sensitive user flow.
  - External AI calls wrapped in a timeout (10s for suggestions, 5s for tone adjustment) with graceful degradation — if AI is unavailable, return `408 Request Timeout` with instructions to retry.
  - SEO metadata: if user has manually set any SEO field (title/description/keywords), the auto-generation is skipped for that field (FR-006: "保留其手動設定").
  - AI integration point: single adapter interface (`IAiProvider`) with a concrete implementation for the chosen provider. Swappable without touching business logic.

### 6.5 Frontend SPA (網頁介面)
- **Responsibility:** Article editor, media library, user management UI, authentication flows.
- **Serves:** All FRs (presentation layer)
- **Interfaces:**
  - Outbound: All `api/v1/*` endpoints (single source of truth for backend communication)
  - Inbound: Browser HTTP requests (static asset serving from backend or CDN)
- **Data it owns / touches:** None (stateless, communicates via API)
- **Applicable NFRs:**
  - NFR-001: Article editor page <2s load — achieved via initial data prefetch on page load (articles, categories, tags, media list), not lazy-loading.
  - NFR-003: SPA served from static assets; backend handles dynamic API calls only.
- **Depends on:** Backend API (all data access)
- **Key decisions:**
  - SPA framework: React (or equivalent) with server-side rendering for initial page loads to meet NFR-001.
  - State management: client-side cache with optimistic updates for editor; refetch on status change.
  - Authentication: JWT stored in httpOnly cookie (not localStorage) to prevent XSS token theft.
  - Editor: rich-text Markdown editor with inline AI suggestion panel (FR-007, FR-008).

## 7. Technology Decisions

| Decision | Choice | Rationale | Rejected alternative |
|---|---|---|---|
| Backend framework | Node.js + Express (or NestJS) | Matches existing team stack assumption; fast iteration; ecosystem for image processing (sharp), JWT, validation. | Python/Django — heavier for this scope; Ruby on Rails — smaller ecosystem for AI integration. |
| Database | PostgreSQL | ACID compliance for RBAC and article state transitions; JSONB for SEO keywords; mature ecosystem. | SQLite — insufficient for 500 concurrent users (NFR-001). MongoDB — no ACID for transactional auth operations. |
| Session / Auth | JWT (access) + refresh token (httpOnly cookie) | Stateless access tokens for API; refresh tokens in cookies prevent XSS token theft. | Server-side sessions — unnecessary state at this scale; localStorage tokens — XSS vulnerable. |
| Password hashing | bcrypt (cost factor 12) | FR-004 requires salted hashing; bcrypt is battle-tested and meets NFR-002. | Argon2id — better security but higher CPU cost at 500 concurrent logins; PBKDF2 — weaker against GPU attacks. |
| File storage | Local filesystem (uploads/ with hashed directories) | Simple, fast for single-server deployment at 10K DAU. | S3/cloud storage — overkill for current scale; adds operational complexity. |
| Thumbnail generation | sharp (Node.js image processing) | Fast, compiled, zero-dependency on system libraries. | GraphicsMagick/ImageMagick — heavier system dependency. |
| AI integration | Adapter pattern with pluggable provider | FR-007/008 specify "TBD provider"; adapter allows swap without code changes. | Hard-coded provider call — locks us in before provider is chosen. |
| Frontend | React SPA with SSR | Meets NFR-001 initial load time; large ecosystem for rich-text editors and AI UI patterns. | Server-rendered HTML templates — harder to maintain interactive editor state. |

## 8. How Non-Functional Requirements Are Met

| NFR | How the design meets it |
|---|---|
| NFR-001 (performance: editor page <2s at 500 concurrent users, p95) | Single monolith eliminates cross-service latency. Article editor data prefetched in single API call. PostgreSQL indexes on `status`, `author_id`, `category_id`. SSR for initial page load. Redis optional for caching frequently-accessed article data if benchmarks show p95 >2s. |
| NFR-002 (security: password hashing, PII encrypted) | bcrypt cost 12 for all passwords. TLS for all traffic. Password hashes stored as salted values only. JWT refresh tokens in httpOnly cookies. PII (email, reset tokens) encrypted at rest via PostgreSQL pgcrypto or application-level encryption. GDPR compliance: data export/deletion endpoints, consent logging. |
| NFR-003 (availability: 99.9% during business hours) | Monolith deployment on single container with health check. File backups for uploads. Database daily backup. Graceful degradation: if AI service is down, editor still functions (SEO/manual content). Load testing at 500 concurrent users before production. |
| NFR-004 (scalability: 10,000 DAU) | PostgreSQL connection pooling (PgBouncer if needed). Static media served via CDN (optional). Stateless API design allows horizontal scaling of backend if needed. No session affinity required (JWT-based auth). |

## 9. Risks & Trade-offs

- **Risk:** External AI service provider not chosen yet (PRD §10 open question) — blocks Content Intelligence implementation. **Mitigation:** Adapter interface designed to be provider-agnostic; implementation can use a mock during development.
- **Risk:** Local file storage doesn't scale beyond single server. **Mitigation:** Adapter pattern for storage (like AI adapter); swap to S3-compatible storage without changing business logic when needed.
- **Risk:** Monolith may become hard to maintain if scope grows. **Mitigation:** Clean layered architecture (controllers → services → repositories) enforces boundaries that make future decomposition easier.
- **Risk:** GDPR compliance requires data export/deletion — not explicitly required in FRs. **Mitigation:** Design Users entity with soft-delete and export capability; GDPR is a hard constraint (§8 of PRD), so it must be implemented even if not enumerated as FR.

## 10. Open Questions

- **Who answers:** Planner / Foan
  - What is the specific external AI service provider? (PRD §10: "specific provider TBD") — affects adapter implementation details.
- **Who answers:** Planner / Foan
  - What are the specific regional data security requirements beyond GDPR? (PRD §10: GDPR/CCPA) — affects encryption scope and data residency decisions.
- **Who answers:** Architect (self-resolved)
  - SSR framework choice for frontend (Next.js vs. custom SSR) — will be decided during implementation based on team familiarity.

---

_Approval at Gate 2 authorizes creation of the implementation tasks defined in the accompanying **Task Breakdown**. This document and the Task Breakdown are approved together._
