# 任務分解：OU Know

> **撰寫規範（Architect 必須遵守）：**
> 1. 這是實作工作的權威清單。Jarvis 在 Gate 2 會原樣轉錄至 `WorkTask.md` —— 它不會增、刪、或重新解讀。這裡沒有的就不會建。
> 2. 每個任務都追溯至至少一個 FR（或標記 `enabling` 表示無直接 FR 的基礎設施工作）。無 orphan 任務。
> 3. 你定義以下欄位。Jarvis 從 workflow rules 填入其餘欄位（`State`、`Status`、`Priority`、`Rejects`）—— 不要在此設定。
> 4. 明確定義 dependency 和 subtask 圖。`Depends On` = 執行順序；`SubtaskOf` = 上層 parent。任務不得依賴自己的 parent。
> 5. 每個可執行任務指派給 exactly one owner：`dev-be` 或 `dev-fe`。Parent（roll-up）任務無 owner。
> _(提交前刪除此區塊。)_

## Meta

| 欄位 | 值 |
|---|---|
| ProjectId | PJ-0002 |
| 作者 | Architect |
| 基於架構版本 | 0.1 |
| 日期 | 2026-09-01 |
| 狀態 | Draft |

---

## Tasks

| Work | 標題 | Owner | Reviewer | Depends On | SubtaskOf | Traces to | Reads |
|------|------|-------|----------|------------|-----------|-----------|-------|
| T-010 | 專案初始化與資料庫設定 | - | - | - | - | enabling | Arch §2 |
| T-011 | 初始化 backend 專案（Express/NestJS，分層結構） | dev-be | reviewer | T-010 | T-010 | enabling | Arch §2 |
| T-012 | 初始化 frontend 專案（React SPA + SSR） | dev-fe | reviewer | T-010 | T-010 | enabling | Arch §6.5 |
| T-013 | 資料庫 schema 與 migration（PostgreSQL） | dev-be | reviewer | T-010 | T-010 | enabling | Arch §5 |
| T-020 | 認證系統 | - | - | - | - | FR-003, FR-004, FR-005 | Arch §6.1 |
| T-021 | User model + bcrypt 密碼雜湊 + 註冊 API | dev-be | reviewer | T-013 | T-020 | FR-004, FR-005 | Arch §6.1, §5 |
| T-022 | 登入 API 搭配 lockout 邏輯 | dev-be | reviewer | T-021 | T-020 | FR-003 | Arch §6.1 |
| T-023 | JWT access + refresh token（httpOnly cookie） | dev-be | reviewer | T-022 | T-020 | FR-005 | Arch §6.1 |
| T-024 | 密碼重設流程（request + confirm） | dev-be | reviewer | T-023 | T-020 | FR-003 | Arch §6.1 |
| T-025 | RBAC middleware + route guards | dev-be | reviewer | T-023 | T-020 | FR-005 | Arch §6.1 |
| T-026 | Auth 頁面（login、register、password reset） | dev-fe | reviewer | T-022 | T-020 | FR-003 | Arch §6.5 |
| T-030 | 文章管理 | - | - | - | - | FR-001, FR-002 | Arch §6.2 |
| T-031 | Article model + category/tag models + junction tables | dev-be | reviewer | T-013 | T-030 | FR-001 | Arch §5 |
| T-032 | 文章 CRUD API（GET/POST/PUT/DELETE） | dev-be | reviewer | T-031, T-025 | T-030 | FR-001 | Arch §6.2 |
| T-033 | 文章狀態轉換邏輯（draft/review/published/archived） | dev-be | reviewer | T-032 | T-030 | FR-002 | Arch §6.2 |
| T-034 | 文章 publish/revert endpoints | dev-be | reviewer | T-033 | T-030 | FR-002 | Arch §6.2 |
| T-035 | 文章編輯器頁面（表單、存草稿、發布） | dev-fe | reviewer | T-032 | T-030 | FR-001 | Arch §6.5 |
| T-036 | 文章列表/詳細頁面 | dev-fe | reviewer | T-032 | T-030 | FR-001 | Arch §6.5 |
| T-040 | 媒體管理 | - | - | - | - | FR-003 (media) | Arch §6.3 |
| T-041 | Media model + 上傳驗證（類型 + 大小） | dev-be | reviewer | T-013 | T-040 | FR-003 (media) | Arch §5, §6.3 |
| T-042 | 媒體上傳 API + 縮圖產生 | dev-be | reviewer | T-041 | T-040 | FR-003 (media) | Arch §6.3 |
| T-043 | 媒體庫頁面（列表、預覽、關聯至文章） | dev-fe | reviewer | T-042 | T-040 | FR-003 (media) | Arch §6.5 |
| T-050 | 內容智慧 | - | - | - | - | FR-006, FR-007, FR-008 | Arch §6.4 |
| T-051 | AI provider adapter interface + mock 實現 | dev-be | reviewer | T-011 | T-050 | FR-007 | Arch §6.4 |
| T-052 | SEO metadata 產生 API | dev-be | reviewer | T-051, T-032 | T-050 | FR-006 | Arch §6.4 |
| T-053 | AI 寫作建議 API | dev-be | reviewer | T-051 | T-050 | FR-007 | Arch §6.4 |
| T-054 | 語氣調整 API | dev-be | reviewer | T-051 | T-050 | FR-008 | Arch §6.4 |
| T-055 | 編輯器中的 AI suggestion panel + 語氣調整 UI | dev-fe | reviewer | T-053, T-054 | T-050 | FR-006, FR-007, FR-008 | Arch §6.5 |
| T-060 | 管理者使用者管理 | - | - | - | - | FR-005 | Arch §6.1 |
| T-061 | 使用者管理 API（CRUD users、roles） | dev-be | reviewer | T-021, T-025 | T-060 | FR-005 | Arch §5, §6.1 |
| T-062 | 使用者管理頁面（列表、建立、編輯、刪除使用者） | dev-fe | reviewer | T-061 | T-060 | FR-005 | Arch §6.5 |

## Coverage Check

| 需求 | 對應任務 |
|---|---|
| FR-001（文章建立表單） | T-031, T-032, T-035 |
| FR-002（文章狀態轉換） | T-033, T-034 |
| FR-003（資源上傳驗證） | T-041, T-042 |
| FR-003（使用者登入與密碼管理） | T-021, T-022, T-024, T-026 |
| FR-004（密碼雜湊） | T-021 |
| FR-005（基於角色的存取控制） | T-025, T-061 |
| FR-006（SEO Metadata 生成） | T-052, T-055 |
| FR-007（AI 內容寫作輔助） | T-053, T-055 |
| FR-008（語氣調整工具） | T-054, T-055 |
| NFR-001（效能） | Arch §8（monolith + indexing + SSR） |
| NFR-002（安全） | Arch §8（bcrypt + TLS + JWT cookie） |
| NFR-003（可用性） | Arch §8（health check + backups + degradation） |
| NFR-004（延展性） | Arch §8（connection pooling + stateless API） |

---

_於 Gate 2 核准後，Jarvis 會在 `WorkTask.md` 中建立 exactly 這些任務，解決 dependency，將 unblocked 任務設為 `Ready`，並通知其 owner。_
