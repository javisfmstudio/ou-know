# Task Breakdown: OU Know

> **Authoring rules (Architect must follow):**
> 1. This is the authoritative list of implementation work. Jarvis transcribes it into `WorkTask.md` verbatim at Gate 2 — it does not add, remove, or reinterpret. What isn't here does not get built.
> 2. Every task traces to at least one FR (or is marked `enabling` for infrastructure with no direct FR). No orphan tasks.
> 3. You define the columns below. Jarvis fills the rest (`State`, `Status`, `Priority`, `Rejects`) from workflow rules — do not set those here.
> 4. Make the dependency and subtask graph explicit and acyclic. `Depends On` = execution order; `SubtaskOf` = roll-up parent. A task must not depend on its own parent.
> 5. Assign each executable task to exactly one owner: `dev-be` or `dev-fe`. Parent (roll-up) tasks have no owner.
> _(Delete this block before submitting.)_

## Meta

| Field | Value |
|---|---|
| ProjectId | PJ-0002 |
| Author | Architect |
| Based on Architecture version | 0.1 |
| Date | 2026-09-01 |
| Status | Draft |

---

## Tasks

| Work | Title | Owner | Reviewer | Depends On | SubtaskOf | Traces to | Reads |
|------|-------|-------|----------|------------|-----------|-----------|-------|
| T-010 | Project scaffolding & database setup | - | - | - | - | enabling | Arch §2 |
| T-011 | Initialize backend project (Express/NestJS, layered structure) | dev-be | reviewer | T-010 | T-010 | enabling | Arch §2 |
| T-012 | Initialize frontend project (React SPA + SSR) | dev-fe | reviewer | T-010 | T-010 | enabling | Arch §6.5 |
| T-013 | Database schema & migration (PostgreSQL) | dev-be | reviewer | T-010 | T-010 | enabling | Arch §5 |
| T-020 | Authentication system | - | - | - | - | FR-003, FR-004, FR-005 | Arch §6.1 |
| T-021 | User model + bcrypt password hashing + registration API | dev-be | reviewer | T-013 | T-020 | FR-004, FR-005 | Arch §6.1, §5 |
| T-022 | Login API with lockout logic | dev-be | reviewer | T-021 | T-020 | FR-003 | Arch §6.1 |
| T-023 | JWT access + refresh token (httpOnly cookie) | dev-be | reviewer | T-022 | T-020 | FR-005 | Arch §6.1 |
| T-024 | Password reset flow (request + confirm) | dev-be | reviewer | T-023 | T-020 | FR-003 | Arch §6.1 |
| T-025 | RBAC middleware + route guards | dev-be | reviewer | T-023 | T-020 | FR-005 | Arch §6.1 |
| T-026 | Auth pages (login, register, password reset) | dev-fe | reviewer | T-022 | T-020 | FR-003 | Arch §6.5 |
| T-030 | Article management | - | - | - | - | FR-001, FR-002 | Arch §6.2 |
| T-031 | Article model + category/tag models + junction tables | dev-be | reviewer | T-013 | T-030 | FR-001 | Arch §5 |
| T-032 | Article CRUD API (GET/POST/PUT/DELETE) | dev-be | reviewer | T-031, T-025 | T-030 | FR-001 | Arch §6.2 |
| T-033 | Article status transition logic (draft/review/published/archived) | dev-be | reviewer | T-032 | T-030 | FR-002 | Arch §6.2 |
| T-034 | Article publish/revert endpoints | dev-be | reviewer | T-033 | T-030 | FR-002 | Arch §6.2 |
| T-035 | Article editor page (form, save draft, publish) | dev-fe | reviewer | T-032 | T-030 | FR-001 | Arch §6.5 |
| T-036 | Article list / detail pages | dev-fe | reviewer | T-032 | T-030 | FR-001 | Arch §6.5 |
| T-040 | Media management | - | - | - | - | FR-003 (media) | Arch §6.3 |
| T-041 | Media model + upload validation (type + size) | dev-be | reviewer | T-013 | T-040 | FR-003 (media) | Arch §5, §6.3 |
| T-042 | Media upload API + thumbnail generation | dev-be | reviewer | T-041 | T-040 | FR-003 (media) | Arch §6.3 |
| T-043 | Media library page (list, preview, associate with article) | dev-fe | reviewer | T-042 | T-040 | FR-003 (media) | Arch §6.5 |
| T-050 | Content Intelligence | - | - | - | - | FR-006, FR-007, FR-008 | Arch §6.4 |
| T-051 | AI provider adapter interface + mock implementation | dev-be | reviewer | T-011 | T-050 | FR-007 | Arch §6.4 |
| T-052 | SEO metadata generation API | dev-be | reviewer | T-051, T-032 | T-050 | FR-006 | Arch §6.4 |
| T-053 | AI writing suggestion API | dev-be | reviewer | T-051 | T-050 | FR-007 | Arch §6.4 |
| T-054 | Tone adjustment API | dev-be | reviewer | T-051 | T-050 | FR-008 | Arch §6.4 |
| T-055 | AI suggestion panel + tone adjust UI in editor | dev-fe | reviewer | T-053, T-054 | T-050 | FR-006, FR-007, FR-008 | Arch §6.5 |
| T-060 | Admin user management | - | - | - | - | FR-005 | Arch §6.1 |
| T-061 | User management API (CRUD users, roles) | dev-be | reviewer | T-021, T-025 | T-060 | FR-005 | Arch §5, §6.1 |
| T-062 | User management page (list, create, edit, delete users) | dev-fe | reviewer | T-061 | T-060 | FR-005 | Arch §6.5 |

## Coverage Check

| Requirement | Covered by |
|---|---|
| FR-001 (文章建立表單) | T-031, T-032, T-035 |
| FR-002 (文章狀態轉換) | T-033, T-034 |
| FR-003 (資源上傳驗證) | T-041, T-042 |
| FR-003 (使用者登入與密碼管理) | T-021, T-022, T-024, T-026 |
| FR-004 (密碼雜湊) | T-021 |
| FR-005 (基於角色的存取控制) | T-025, T-061 |
| FR-006 (SEO Metadata 生成) | T-052, T-055 |
| FR-007 (AI 內容寫作輔助) | T-053, T-055 |
| FR-008 (語氣調整工具) | T-054, T-055 |
| NFR-001 (效能) | Arch §8 (monolith + indexing + SSR) |
| NFR-002 (安全) | Arch §8 (bcrypt + TLS + JWT cookie) |
| NFR-003 (可用性) | Arch §8 (health check + backups + degradation) |
| NFR-004 (延展性) | Arch §8 (connection pooling + stateless API) |

---

_On Gate 2 approval, Jarvis creates exactly these tasks in `WorkTask.md`, resolves dependencies, sets unblocked tasks to `Ready`, and notifies their owners._
