# 任務分解：OU Know v2（整合 Payload 網站模板 + PJ-0002 內容平台）

> **撰寫規範（Architect 必須遵守）：**
> 1. 這是實作工作的權威清單。Jarvis 在 Gate 2 會原樣轉錄至 `WorkTask.md` —— 它不會增、刪、或重新解讀。這裡沒有的就不會建。
> 2. 每個任務都追溯至至少一個 FR（或標記 `enabling` 表示無直接 FR 的基礎設施工作）。無 orphan 任務。
> 3. 你定義以下欄位。Jarvis 從 workflow rules 填入其餘欄位（`State`、`Status`、`Priority`、`Rejects`）—— 不要在此設定。
> 4. 明確定義 dependency 和 subtask 圖。`Depends On` = 執行順序；`SubtaskOf` = 上層 parent。任務不得依賴自己的 parent。
> 5. 每個可執行任務指派給 exactly one owner：`dev-be` 或 `dev-fe`。Parent（roll-up）任務無 owner。
> _(提交前刪此地塊。)_

## Meta

| 欄位 | 值 |
|---|---|
| ProjectId | PJ-0002 |
| 作者 | Architect |
| 基於架構版本 | 0.2 |
| 日期 | 2026-09-05 |
| 狀態 | Draft |

---

## Tasks

| Work | 標題 | Owner | Reviewer | Depends On | SubtaskOf | Traces to | Reads |
|------|-------|-------|----------|------------|-----------|-----------|-------|
| T-100 | 專案初始化與資料庫設定 | - | - | - | - | enabling | Arch §2 |
| T-101 | 初始化 Payload 3.x monolith 專案（自訂，分層結構） | dev-be | reviewer | T-100 | T-100 | enabling | Arch §2 |
| T-102 | 初始化 frontend 專案（Next.js App Router + Payload Admin） | dev-fe | reviewer | T-100 | T-100 | enabling | Arch §6.12 |
| T-103 | 資料庫 schema 與 migration（PostgreSQL，Payload SQL adapter） | dev-be | reviewer | T-100 | T-100 | enabling | Arch §5 |
| T-200 | 認證與 RBAC 系統 | - | - | - | - | FR-003, FR-004, FR-005 | Arch §6.1 |
| T-201 | Users 集合擴充：role、loginAttempts、lockedUntil + bcrypt 密碼杂湊 | dev-be | reviewer | T-103 | T-200 | FR-004, FR-005 | Arch §5.1, §6.1 |
| T-202 | 登入 API（`/api/v1/auth/login`）搭配 lockout 邏輯 | dev-be | reviewer | T-201 | T-200 | FR-003 | Arch §4.1, §6.1 |
| T-203 | JWT access + refresh token（httpOnly cookie） | dev-be | reviewer | T-202 | T-200 | FR-005 | Arch §6.1 |
| T-204 | 註冊 API（`/api/v1/auth/register`） dev-be | dev-be | reviewer | T-201 | T-200 | FR-005 | Arch §4.1, §6.1 |
| T-205 | 密碼重設流程（request + confirm，`/api/v1/auth/password-reset/*`） | dev-be | reviewer | T-203 | T-200 | FR-003 | Arch §4.1, §6.1 |
| T-206 | RBAC access control 規則（src/access/*，三角色權限） | dev-be | reviewer | T-201 | T-200 | FR-005 | Arch §6.1 |
| T-207 | Auth 頁面（login、register、password reset UI） | dev-fe | reviewer | T-202 | T-200 | FR-003 | Arch §6.12 |
| T-300 | Posts 文章管理 + PJ-0002 狀態流 | - | - | - | - | FR-001, FR-002 | Arch §6.2 |
| T-301 | Posts 集合設定：狀態機欄位（_status、reviewStatus、reviewedBy、reviewedAt）+ slug + 分類/標籤/作者 | dev-be | reviewer | T-103 | T-300 | FR-001 | Arch §5.1, §6.2 |
| T-302 | Posts collection beforeChange hook（狀態轉換強制執行） | dev-be | reviewer | T-301 | T-300 | FR-002 | Arch §6.2 |
| T-303 | 文章 CRUD API（`/api/v1/articles[/:id]`） | dev-be | reviewer | T-301, T-206 | T-300 | FR-001 | Arch §4.1, §6.2 |
| T-304 | 文章 publish / to-review / approve / reject / archive endpoints（審流） | dev-be | reviewer | T-302 | T-300 | FR-002 | Arch §4.1, §6.2 |
| T-305 | 文章列表/詳細頁面（Next.js frontend） | dev-fe | reviewer | T-303 | T-300 | FR-001 | Arch §6.2, §6.12 |
| T-306 | 文章編輯器頁面（表單、存草稿、提交審批、發布） | dev-fe | reviewer | T-303 | T-300 | FR-001, FR-002 | Arch §6.2, §6.12 |
| T-400 | Pages 靜態頁面 + Layout Builder | - | - | - | - | enabling, NFR-001 | Arch §6.3 |
| T-401 | Pages 集合設定：layout blocks（CallToAction/Content/MediaBlock/Archive/FormBlock）+ hero | dev-be | reviewer | T-103 | T-400 | enabling | Arch §5.1, §6.3 |
| T-402 | Pages frontend 渲染（Next.js，含 Layout Builder 區塊渲染） | dev-fe | reviewer | T-401 | T-400 | enabling, NFR-001 | Arch §6.3, §6.12 |
| T-500 | 媒體資源管理 + 版本控制 | - | - | - | - | FR-003, UF-002 | Arch §6.4 |
| T-501 | Media 集合設定 + 上傳驗證（類型 + 大小） | dev-be | reviewer | T-103 | T-500 | FR-003 | Arch §5.1, §6.4 |
| T-502 | 媒體上傳 API（`/api/v1/media/upload`）+ 縮圖產生 | dev-be | reviewer | T-501 | T-500 | FR-003 | Arch §4.1, §6.4 |
| T-503 | 媒體版本控制（media_versions 集合 + 版本歷史） | dev-be | reviewer | T-501 | T-500 | FR-003 | Arch §5.2, §6.4 |
| T-504 | 媒體庫頁面（列表、預覽、關聯至文章） | dev-fe | reviewer | T-502 | T-500 | FR-003 | Arch §6.4, §6.12 |
| T-505 | 文章-媒體關聯 endpoint（`/api/v1/articles/:id/media`） | dev-be | reviewer | T-502, T-303 | T-500 | UF-002 | Arch §4.1, §6.2 |
| T-600 | Categories 分類管理 | - | - | - | - | FR-001, UF-002 | Arch §6.5 |
| T-601 | Categories 集合設定（nested docs 巢狀分類） | dev-be | reviewer | T-103 | T-600 | FR-001 | Arch §5.1, §6.5 |
| T-602 | 分類 API（`/api/v1/categories`）+ 前端分類選單 | dev-fe | reviewer | T-601 | T-600 | FR-001 | Arch §4.1, §6.5 |
| T-700 | Content Intelligence（AI 功能）| - | - | - | - | FR-006, FR-007, FR-008 | Arch §6.6 |
| T-701 | AI Provider Adapter（IAiProvider 介面 + mock 實現） | dev-be | reviewer | T-103 | T-700 | FR-007 | Arch §6.6 |
| T-702 | SEO metadata 生成 API（`/api/v1/articles/:id/seo/generate`） | dev-be | reviewer | T-701, T-303 | T-700 | FR-006 | Arch §4.1, §6.6 |
| T-703 | AI 寫作建議 API（`/api/v1/articles/:id/ai/suggest`） | dev-be | reviewer | T-701 | T-700 | FR-007 | Arch §4.1, §6.6 |
| T-704 | 語氣調整 API（`/api/v1/articles/:id/tone/adjust`） | dev-be | reviewer | T-701 | T-700 | FR-008 | Arch §4.1, §6.6 |
| T-705 | 編輯器中的 AI suggestion panel + 語氣調整 UI | dev-fe | reviewer | T-703, T-704 | T-700 | FR-006, FR-007, FR-008 | Arch §6.6, §6.12 |
| T-800 | AI Audit（審流與生成記錄）| - | - | - | - | FR-002, FR-006, NFR-002 | Arch §6.7 |
| T-801 | AI_Audit 集合設定（審批記錄） | dev-be | reviewer | T-103 | T-800 | FR-002, NFR-002 | Arch §5.2, §6.7 |
| T-802 | AI_Generation_Records 集合設定（AI 生成記錄） | dev-be | reviewer | T-103 | T-800 | FR-006, FR-007, FR-008 | Arch §5.2, §6.7 |
| T-803 | SEO_History 集合設定（SEO 元資料歷史） | dev-be | reviewer | T-103 | T-800 | FR-006 | Arch §5.2, §6.7 |
| T-804 | 審流 hook：reviewStatus 變更時寫入 AI_Audit | dev-be | reviewer | T-801, T-302 | T-800 | FR-002 | Arch §6.7 |
| T-805 | AI 呼叫 hook：AI_Generation_Records 寫入（T-701/T-702/T-703） | dev-be | reviewer | T-802, T-701 | T-800 | FR-006, FR-007, FR-008 | Arch §6.7 |
| T-900 | Redirects（URL 重導向）| - | - | - | - | enabling, NFR-001 | Arch §6.8 |
| T-901 | redirectsPlugin 設定 + 重導向 API（`/api/v1/redirects`） | dev-be | reviewer | T-103 | T-900 | enabling | Arch §5.2, §6.8 |
| T-1000 | Search（全文搜尋）| - | - | - | - | enabling, NFR-001 | Arch §6.9 |
| T-1001 | searchPlugin 設定 + 搜尋 API（`/api/v1/search`） | dev-be | reviewer | T-103 | T-1000 | enabling | Arch §5.2, §6.9 |
| T-1100 | Revalidation & Jobs（On-demand Revalidation + Scheduled Publishing）| - | - | - | - | enabling, NFR-001, NFR-003 | Arch §6.10 |
| T-1101 | on-demand revalidation hooks（afterChange → Next.js revalidatePath/Tag） | dev-be | reviewer | T-301, T-401 | T-1100 | enabling, NFR-001 | Arch §6.10 |
| T-1102 | Scheduled Publishing（versions.drafts.schedulePublish + Jobs 佇列） | dev-be | reviewer | T-301 | T-1100 | enabling, NFR-003 | Arch §6.10 |

## Coverage Check

| 需求 | 對應任務 |
|---|---|
| FR-001（文章建立表單）| T-301, T-303, T-305, T-306, T-601 |
| FR-002（文章狀態轉換 + 審批流）| T-302, T-304, T-804 |
| FR-003（資源上傳驗證 + 媒體版本控制）| T-501, T-502, T-503, T-505 |
| FR-003（使用者登入與密碼管理）| T-201, T-202, T-204, T-205, T-207 |
| FR-004（密碼杂湊）| T-201 |
| FR-005（基於角色的存取控制）| T-201, T-206, T-207 |
| FR-006（SEO Metadata 生成）| T-702, T-803, T-805 |
| FR-007（AI 內容寫作輔助）| T-701, T-703, T-805, T-705 |
| FR-008（語氣調整工具）| T-701, T-704, T-705 |
| NFR-001（效能：編輯器 <2s）| Arch §8（monolith + indexing + SSR + revalidation）；T-305, T-402, T-1101 |
| NFR-002（安全：密码杂湊、PII 加密、audit trail）| Arch §8（bcrypt + TLS + JWT cookie + AI Audit）；T-201, T-801, T-802, T-803 |
| NFR-003（可用性：99.9% uptime）| Arch §8（health check + backups + degradation）；T-1102 |
| NFR-004（延展性：10,000 DAU）| Arch §8（connection pooling + stateless API）；T-103 |
| 網站面功能（Pages、Layout Builder、Redirects、Search、Draft/Live Preview、Jobs、Scheduled Publishing、Auth/Access Control）| T-401, T-402, T-901, T-1001, T-1101, T-1102, T-206 |

---

_於 Gate 2 核准後，Jarvis 會在 `WorkTask.md` 中建立 exactly 這些任務，解決 dependency，將 unblocked 任務設為 `Ready`，並通知其 owner。_
