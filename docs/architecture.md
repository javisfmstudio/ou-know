# 架構設計：OU Know v2（整合 Payload 網站模板 + PJ-0002 內容平台）

> **撰寫規範（Architect 必須遵守）：**
> 1. 本文回答 PRD 未開的 **how**。不重新開啟或重新協商需求 —— 若需求有誤或不清楚，提出澄清，不要靜默重新設計。
> 2. 每個設計決策都追溯至需求（FR-XXX / NFR-XXX）。若 component 未對應到任何需求，以 enabling 工作說明或刪除此 component。
> 3. 明確定義 component 之間的 contract —— 特別是 Dev-be / Dev-fe 邊界。兩個 agent 對著模糊介面開發，會做出不相容的兩半。
> 4. 說明 trade-off 與 rejected alternatives。沒有 trade-off 的設計是沒有思考過的設計。
> 5. 不在本文分配或排程工作 —— 那是 Task Breakdown 的事。本文是「它長什麼樣子」，不是「誰做什麼、何時做」。
> _(提交前刪此地塊。)_

## Meta

| 欄位 | 值 |
|---|---|
| ProjectId | PJ-0002 |
| 作者 | Architect |
| 日期 | 2026-09-05 |
| 版本 | 0.2 |
| 基於 PRD 版本 | 0.1 |
| Gate 2 核准者 | Foan / Mike |
| 狀態 | Draft |
| 取代 | v0.1（commit 62fc77d，自訂 monolith Node.js+PostgreSQL）|

---

## 1. Approach

OU Know v2 是一個統一的數位內容管理平台，整合兩個來源的功能面：`ouknow_website`（Payload 3.x Website Template）的完整 CMS 能力（Posts、Pages、Media、Categories、Projects、SEO、Search、Redirects、Draft/Live Preview、On-demand Revalidation、Jobs & Scheduled Publishing、Layout Builder、Auth/Access Control），以及原 PJ-0002 PRD 的內容管理與 AI 驅動功能（文章 CRUD + 狀態流、分類標籤、媒體版本控制、帳號與 RBAC、SEO metadata 生成、AI 輔助寫作、語氣調整）。

解決方案為**在 Payload 3.x 自訂 monolith 之上的「API 客製化 + 內建管理後台」**：Payload 已內建認證、RBAC、GraphQL/REST API、文檔版本與草稿、內容智慧外掛（SEO/Search/Redirects）、Jobs 佇列、Layout Builder，我們在此之上擴充表結構、狀態機、AI 審批流與自有 REST API，而非推倒重寫。最重要的設計決策是**將「PJ-0002 的狀態流（draft/review/published/archived）與審批流」轉譯為 Payload 的 `draft` 欄位與 custom `_review` 欄位**，並明確區分「哪些功能走 Payload 內建 GraphQL/REST、哪些是我們自訂的 REST API endpoint」—— 這是自有 API 需求與 Payload 既有 API 之間的邊界。

## 2. 架構風格與規範

**風格：Monolith（自訂 Payload，已論證）**

| 面向 | 決策 | 為何適合此專案規模 |
|---|---|---|
| 架構風格 | Monolith（自訂 Payload 3.x，單一 Node.js 程序服務 API + Admin 後台 + Next.js frontend）| 功能面雖廣（兩個來源整合），但單一內容數據層、單一資料庫、單一身份層、四個緊密耦合領域（文章/資源/帳號/AI）。無獨立部署需求。Payload monolith 消除了跨服務延遲、分散式交易複雜度與營運 overhead，同時已內建我們在 PRD 中列出的絕大多數後端能力。 |

**所有程式碼必須遵守的規範：**
- **Clean code 基線（不可妥協）：** 清晰的命名、單一職責函式、無死碼或重複程式碼、可讀的結構。Code only a machine can maintain is a defect。
- **程式碼組織（Payload 慣例，開發者必須遵守）：** `src/collections/*`（資料集合設定，對應一張資料表）→ `src/collections/<Name>/hooks/*`（集合級 hooks：beforeChange/afterChange/afterRead，強制狀態機與資料一致性）→ `src/access/*`（跨集合存取控制規則）→ `src/plugins/*`（功能外掛：SEO/Search/Redirects 等）→ `src/fields/*`、`src/blocks/*`（可重複使用的 field/block 定義）→ `src/utilities/*`（純函式工具）。`src/endpoints/*`（自訂 API routes，包含 `/api/seed`）。禁止在 collection 設定中直接寫入複雜 business logic——抽成 hooks。
- **命名規範：** collection slug 用 kebab-case（`posts`、`pages`、`ai-generation-records`）；field 名用 camelCase；type 用 Payload 標準字首（`text`、`richText`、`relationship`、`upload`、`select`、`number`、`date`、`array`、`blocks`、`group`）；REST endpoint 路徑 `/api/<resource>`（保留版本化區段見 §4）。
- **錯誤處理：** 所有錯誤遵循結構化 `ApiError` 格式 `{code, message, details}`。Payload 的 `payload.logger` 用於伺服器端日誌；raw exception 不得洩漏至 HTTP response。自訂 endpoint 必須包 `try/catch` 並回傳結構化錯誤。
- **狀態機強制：** 所有狀態轉換（文章 `_status`、審批 `reviewStatus`）必須在 collection `hooks.beforeChange` 中強制，不只靠 DB constraint。
- **無 orphan field：** 每個 field 必須對應到某個 FR/NFR 或明確標記 enabling。

## 3. Component Map _(global view)_

| Component | 職責 | 服務於 | 詳細 |
|---|---|---|---|
| Auth & Users | 使用者認證、密碼管理、RBAC 執行 | FR-003, FR-004, FR-005, NFR-002 | §6.1 |
| Posts（文章）| 文章 CRUD、PJ-0002 狀態流（draft/review/published/archived）、分類、標籤、作者、SEO | FR-001, FR-002, UF-001 | §6.2 |
| Pages | 靜態頁面 CRUD、Layout Builder、SEO、導航 | enabling（網站面）, NFR-001 | §6.3 |
| Media（數位資源）| 上傳驗證、儲存、縮圖、版本控制、文章關聯 | FR-003, UF-002, NFR-003 | §6.4 |
| Categories | 分類巢狀結構（nested docs） | FR-001, UF-002 | §6.5 |
| Content Intelligence（AI）| SEO metadata 生成、AI 輔助寫作、語氣調整、AI 審批流 | FR-006, FR-007, FR-008 | §6.6 |
| AI Audit（審批與生成記錄）| 審流（review workflow）、AI 生成記錄、SEO 歷史 | FR-002, FR-006, NFR-002 | §6.7 |
| Redirects | URL 重導向管理 | enabling（網站面）, NFR-001 | §6.8 |
| Search | 文章全文搜尋 | enabling（網站面）, NFR-001 | §6.9 |
| Revalidation & Jobs | On-demand revalidation、Scheduled Publishing、Jobs 佇列 | enabling（網站面）, NFR-001, NFR-003 | §6.10 |
| Layout Builder | 頁面區塊編排器 | enabling（網站面） | §6.3 |
| Frontend SPA | 管理後台 UI + 網站前端（Next.js） | 所有 FR（presentation layer） | §6.11 |

_（圖形化關係見 §3.1）_

### 3.1 關係圖

```
                        ┌─────────────────────────────────────┐
                        │         Frontend（Next.js）          │
                        │  Admin 後台 UI + 網站前端（SSR）      │
                        └───────────────┬──────────────────────┘
                                        │ GraphQL / REST / REST(自有)
                        ┌───────────────▼──────────────────────┐
                        │      自訂 monolith：Payload 3.x       │
                        │                                       │
  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
  │ Auth/Users│  │  Posts    │  │  Media    │  │ Categories│  │
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  │
        │              │              │              │        │
  ┌─────▼──────────────▼──────────────▼──────────────▼─────┐ │
  │              統一資料庫（PostgreSQL）                    │ │
  │  users · posts · pages · media · categories ·         │ │
  │  ai_audit · ai_generation_records · redirect · search  │ │
  └───────────────────────────────────────────────────────┘ │
                        │        │           │               │
              hooks.beforeChange │        │  plugins(SEO/Search│
              hooks.afterChange  │        │  /Redirects/Jobs)  │
                        │        │                                  │
              ┌─────────▼───────▼──────────▼──────────────────────┐
              │  AI Provider Adapter（IAiProvider，provider-agnostic）│
              └────────────────────────────────────────────────────┘
```

## 4. System-Wide Interfaces _(global view)_

_Contracts that cross component boundaries. Prioritize any Dev-be ↔ Dev-fe boundary._

### 4.1 自有 REST API（Dev-be ↔ Dev-fe 的主要 seam）

> 這是 Foan 明確要求「保留自有 API」的核心交付。我們同時提供：
> (a) Payload 內建的 GraphQL + REST（`/api/*`）—— 供 Admin 後台與 Next.js frontend 使用；
> (b) **自訂的、明確版本化的 REST API**（`/api/v1/*`）—— 供外部消費者與前端 SPA 使用，作為我們對 PRD FR 的顯式回應。

| From → To | Contract（endpoint / event / data shape） | Notes |
|---|---|---|
| Frontend/Client → Backend | `POST /api/v1/auth/register` `{email, password, name, role?}` → `{user, token}` | Password ≥8 字元；role 預設 author（FR-005） |
| Frontend/Client → Backend | `POST /api/v1/auth/login` `{email, password}` → `{user, token}` | 失敗 401；5 次失敗 lockout（FR-003） |
| Frontend/Client → Backend | `POST /api/v1/auth/logout` → `{204}` | 清除 httpOnly cookie |
| Frontend/Client → Backend | `POST /api/v1/auth/password-reset/request` `{email}` → `{204}` | 透過 email 發送 reset link（FR-003） |
| Frontend/Client → Backend | `POST /api/v1/auth/password-reset/confirm` `{token, newPassword}` → `{204}` | Token 有效 1 小時，單次使用（FR-003） |
| Frontend/Client → Backend | `GET /api/v1/articles[/:id]` | 文章讀取；published 可見，draft 需 auth（FR-001） |
| Frontend/Client → Backend | `POST /api/v1/articles` `{title, content, categoryId?, tagIds[], status?}` → `{article}` | 建立文章；標題必填（FR-001） |
| Frontend/Client → Backend | `PUT /api/v1/articles/:id` `{partial}` → `{article}` | 編輯文章 |
| Frontend/Client → Backend | `DELETE /api/v1/articles/:id` → `{204}` | 刪除文章（僅 admin/editor） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/publish` → `{article}` | draft → published（FR-002） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/to-review` → `{article}` | draft → review（FR-002 審批流） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/approve` → `{article}` | review → published（FR-002 審批流） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/reject` `{reason}` → `{article}` | review → draft + 記錄拒絕原因（FR-002 審批流） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/archive` → `{article}` | draft → archived（僅 admin） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/media` `{mediaIds[]}` → `{article}` | 將媒體關聯至文章（UF-002） |
| Frontend/Client → Backend | `POST /api/v1/media/upload` (multipart) → `{media, thumbnailUrl}` | 最大 10MB，類型驗證（FR-003） |
| Frontend/Client → Backend | `GET /api/v1/media[/:id]` → `{media}` | 列出 / 取得媒體項目 |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/seo/generate` `{articleId}` → `{seo}` | 自動產生 SEO metadata（FR-006） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/ai/suggest` `{paragraph?, focus?}` → `{suggestions[]}` | AI 寫作輔助（FR-007） |
| Frontend/Client → Backend | `POST /api/v1/articles/:id/tone/adjust` `{paragraph, tone}` → `{rewritten}` | 語氣調整：formal/casual（FR-008） |
| Frontend/Client → Backend | `GET /api/v1/categories` → `{categories}` | 列出分類（含巢狀） |
| Frontend/Client → Backend | `GET /api/v1/search?q=...` → `{results}` | 全文搜尋文章 |
| Frontend/Client → Backend | `GET /api/v1/redirects` → `{redirects}` | 列出重導向設定 |

_所有 endpoint 皆為 `https://`、皆需 JWT（除 login/register/password-reset）。API 版本化以 `/api/v1/` 前綴；未來 major 版本更動用 `/api/v2/`。_

### 4.2 Payload 內建 API（Admin 後台與 Next.js frontend 使用）

| From → To | Contract | Notes |
|---|---|---|
| Admin 後台 → Backend | GraphQL（`/api/graphql`）| Payload 內建 GraphQL schema；Admin 後台用它做 CRUD、filter、populate |
| Next.js frontend → Backend | REST（`/api/posts`、`/api/pages`、`/api/media`、`/api/users/me`）| 動態資料擷取；搭配 `next/headers` 的 draftMode 讀取草稿 |
| Next.js frontend → Backend | GraphQL（`/api/graphql`）| SEO metadata、搜尋、導航資料 |

### 4.3 狀態轉換事件（Component 間 event）

| 事件 | 觸發 | 消費 | Notes |
|---|---|---|---|
| `post._status` 變更 | Posts collection `beforeChange` hook | revalidation hook（§6.10）、AI Audit 記錄 | 統一狀態入口 |
| `post.reviewStatus` 變更 | 審批 endpoints | AI Audit 記錄、revalidation | 審流狀態 |
| `ai_generation_record` 建立 | AI service | 無直接消費者（audit trail）| 生成記錄 |

## 5. Data Model _(global view)_

### 5.1 核心集合（Collections）

#### Users（認證與 RBAC）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key（Postgres adapter）|
| email | Unique text | 登入憑證 |
| password | Hashed | bcrypt/argon2 salted hash（Payload 內建 auth 欄位）|
| name | text | 顯示名稱 |
| role | select(`admin`/`editor`/`author`) | RBAC 角色（FR-005）；**新增**（PRD 要求三角色）|
| resetToken | text | NULL = 無待處理 reset |
| resetTokenExpiry | date | NULL = 無待處理 reset |
| loginAttempts | number | 成功登入後重設為 0（FR-003）|
| lockedUntil | date | NULL = 未 lock |
| createdAt / updatedAt | date | Payload 內建 timestamps |

> Payload 內建 auth collection 已提供 email、password（hashed）、reset token、timestamps。我們在 `Users` 集合上**擴充** `role`、`loginAttempts`、`lockedUntil` 以支援 PRD 的 RBAC 與 lockout。

#### Posts（文章）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| title | text（required）| 標題 |
| content | richText（Lexical，required）| 文章內容（含 blocks）|
| slug | text（unique）| URL-friendly slug |
| `_status` | select(`draft`/`published`/`archived`) | Payload `versions.drafts` 內建 `_status`；**新增** `archived` 選項（PRD FR-002）|
| `reviewStatus` | select(`pending`/`approved`/`rejected`) | **新增**：PJ-0002 審批流（FR-002）|
| `reviewReason` | text | 拒絕原因（reviewStatus=rejected 時）|
| `reviewedBy` | relationship → users | 審批者（**新增**）|
| `reviewedAt` | date | 審批時間（**新增**）|
| categoryId | relationship → categories | 單一分類（PRD FR-001）|
| tagIds | array of text | 標籤（PRD FR-001，簡化自多對多）|
| authors | relationship → users (many) | 作者 |
| meta.title / meta.description / meta.image | group（SEO plugin）| SEO metadata（FR-006）|
| relatedPosts | relationship → posts (many) | 相關文章 |
| heroImage | upload → media | 頭圖 |
| publishedAt | date | 發表時間（Published 時設定）|
| `_createdAt` / `_updatedAt` | date | Payload 內建 |

> **關鍵轉譯：** PRD 的「draft/review/published/archived」四狀態流 = Payload 內建的 `_status`（draft/published/archived）+ 我們新增的 `reviewStatus`（pending/approved/rejected）。`reviewStatus` 在 draft 與 published 之間流動，作為審批門檻。

#### Pages（靜態頁面）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| title | text | 頁面標題 |
| slug | text | URL-friendly（home = 首頁）|
| hero | group（heroType / richText / links / media）| Hero 區塊 |
| layout | blocks（CallToAction/Content/MediaBlock/Archive/FormBlock）| Layout Builder（PRD Layout Builder）|
| meta | group（SEO plugin）| SEO metadata |
| publishedAt | date | 發表時間 |
| `_status` | select | Payload drafts `_status` |

#### Media（數位資源）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| filenameServer | text | 系統產生，唯一 |
| mimetype | text | 上傳時驗證 |
| size | number | 位元組 |
| url | text | 存取 URL |
| thumbnailUrl | text | 縮圖 URL |
| uploadedBy | relationship → users | 上傳者 |
| createdAt | date | 建立時間 |

> **版本控制（PRD「媒體資源管理與版本控制」）：** 透過 Payload `versions.drafts` 或建立 `media_versions` 集合記錄每次上傳的版本歷史。設計見 §6.4。

#### Categories（分類）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| name | text | 分類名稱 |
| slug | text | URL-friendly |
| _children | relationship → categories | 巢狀子分類（nested docs plugin）|
| _level | number | 巢狀層級（nested docs plugin）|

### 5.2 外掛集合（Plugins）

| 集合 | 來源 | 備註 |
|---|---|---|
| redirects | redirectsPlugin | URL 重導向（PRD Redirects）|
| search | searchPlugin | 文章全文搜尋索引（PRD Search）|
| forms, formSubmissions | formBuilderPlugin | 表單與回應（模板內建）|

### 5.3 AI 與審批專屬集合（PRD 額外需求）

#### AI_Audit（審批與內容審查記錄）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| action | select(`create`/`to_review`/`approve`/`reject`/`publish`/`archive`) | 審批動作（PRD 審流）|
| byUser | relationship → users | 執行者 |
| reason | text | 備註/拒絕原因 |
| createdAt | date | 審批時間 |

> 對應 FR-002（審批流）與 NFR-002（稽核日誌）。每次 `reviewStatus` 變更為此表寫入一筆記錄。

#### AI_Generation_Records（AI 生成記錄）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| type | select(`seo`/`suggest`/`tone`) | 生成類型（FR-006/007/008）|
| prompt | text | 原始輸入 |
| response | text | 生成結果 |
| provider | text | 使用的 provider（FR-007 TBD）|
| tokenUsage | number | 使用 token 數（可選）|
| durationMs | number | 處理時間（FR-007：<10s）|
| createdAt | date | 生成時間 |

> 對應 FR-006/007/008 的「生成記錄」需求與 NFR-002（audit trail）。每次 AI 呼叫皆寫入此表。

#### SEO_History（SEO 元資料歷史）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| title | text | 產生的 title |
| description | text | 產生的 description |
| keywords | json | 產生的 keywords |
| generatedBy | select(`ai`/`manual`) | 來源 |
| createdAt | date | 產生時間 |

> 對應 FR-006 與「SEO 元資料」的歷史追蹤需求。

## 6. Component Detail _(per-component — the unit a developer reads)_

### 6.1 Auth & Users（認證與存取控制）
- **Responsibility:** 使用者註冊、登入、密碼管理、RBAC 執行。
- **Serves:** FR-003, FR-004, FR-005, NFR-002
- **Interfaces:**
  - Inbound: `/api/v1/auth/register`, `/api/v1/auth/login`, `/api/v1/auth/logout`, `/api/v1/auth/password-reset/request`, `/api/v1/auth/password-reset/confirm`
  - Outbound: Payload 內建 auth（`/api/users`）、Email service（reset link）
- **Data it owns / touches:** Users 集合（role、loginAttempts、lockedUntil、resetToken）
- **Applicable NFRs:**
  - NFR-002：bcrypt（cost factor 12）用於密碼雜湊；所有 auth traffic 使用 TLS；password hash 以 salted hash 儲存，永不以 plaintext 儲存。
  - NFR-001：登入回應 <100ms p95（indexed email lookup，無重度計算）。
- **Depends on:** Email service（外部）, Frontend SPA（login/register pages）
- **Key decisions:**
  - JWT access tokens（15 min TTL）+ refresh tokens（7 day TTL，以 httpOnly cookies 儲存）。
  - Account lockout：5 次連續失敗 → lock 15 分鐘（FR-003 step 3：「記錄嘗試次數」）。在 `Users` 集合 `beforeChange` hook 中檢查 loginAttempts，在登入 endpoint 中遞增/重設。
  - RBAC：Payload access control（`src/access/*`）按角色執行：admin 全權、editor 可 approve/reject、author 只能編輯自己的文章（FR-005）。

### 6.2 Posts（文章管理 + PJ-0002 狀態流）
- **Responsibility:** 文章 CRUD、PJ-0002 狀態流（draft/review/published/archived）、分類、標籤、作者、審批流。
- **Serves:** FR-001, FR-002, UF-001
- **Interfaces:**
  - Inbound: `/api/v1/articles[/:id]`（CRUD）、`/api/v1/articles/:id/publish`、`/api/v1/articles/:id/to-review`、`/api/v1/articles/:id/approve`、`/api/v1/articles/:id/reject`、`/api/v1/articles/:id/archive`、`/api/v1/articles/:id/media`
  - Outbound: Content Intelligence（SEO auto-fill）、Media（媒體關聯）、AI Audit（審批記錄）
- **Data it owns / touches:** Posts（含 `_status`、`reviewStatus`、`reviewReason`、`reviewedBy`、`reviewedAt`）、Categories、Media
- **Applicable NFRs:**
  - NFR-001：文章編輯器頁面載入 <2s（500 同時使用者）—— 透過文章資料的 server-side rendering 搭配分頁達成，資料庫 on `_status`、`categoryId`、`reviewStatus` 建 index。
  - NFR-004：Schema 支援 10,000 DAU —— posts table 建 index 以利高效過濾。
- **Depends on:** Auth（RBAC）, Content Intelligence（SEO auto-fill）, Media（媒體關聯）, AI Audit（審批記錄）
- **Key decisions:**
  - **狀態機轉譯（關鍵）：** PRD 的 draft/review/published/archived = Payload `_status`（draft/published/archived）+ `reviewStatus`（pending/approved/rejected）。轉換流程：
    - `draft` → `to-review`（author 提交審批）→ `reviewStatus=pending`
    - `pending` → `published`（editor approve）+ `reviewStatus=approved`
    - `pending` → `draft`（editor reject）+ `reviewStatus=rejected` + `reviewReason`
    - `draft` → `published`（直接發表，admin）
    - `published` → `draft`（revert；編輯需先 revert —— FR-002 驗收標準）
    - `draft` → `archived`（僅 admin）
  - 所有轉換在 `Posts` 集合 `beforeChange` hook 中強制，並觸發 AI Audit 寫入記錄。
  - `reviewedBy`、`reviewedAt` 在 approve/reject 時設定。
  - Published 時設定 `publishedAt`，revert 時清除。

### 6.3 Pages（靜態頁面 + Layout Builder）
- **Responsibility:** 靜態頁面 CRUD、Layout Builder、SEO、導航。
- **Serves:** enabling（網站面）, NFR-001
- **Interfaces:**
  - Inbound: Payload 內建 GraphQL（Admin 後台）
  - Outbound: Next.js frontend（SSR 渲染）
- **Data it owns / touches:** Pages（含 layout blocks、hero、meta）
- **Applicable NFRs:**
  - NFR-001：頁面載入透過 Next.js SSG + on-demand revalidation 達成（見 §6.10）。
- **Depends on:** Media（頭圖）, SEO plugin（meta）, Layout Builder（blocks）
- **Key decisions:**
  - Layout Builder 使用 Payload `blocks` 類型，包含 CallToAction、Content、MediaBlock、Archive、FormBlock 區塊（模板內建）。
  - 此 component 為網站面功能，無直接 PRD FR（標記 enabling），但支援 NFR-001（頁面效能）。

### 6.4 Media（數位資源管理 + 版本控制）
- **Responsibility:** 上傳驗證、儲存、縮圖產生、媒體擷取、版本控制。
- **Serves:** FR-003, UF-002, NFR-003
- **Interfaces:**
  - Inbound: `/api/v1/media/upload` (multipart)、`/api/v1/media[/:id]`
  - Outbound: File system / cloud storage（文章上傳）、Image processing library（縮圖產生）
- **Data it owns / touches:** Media 集合（含版本控制）
- **Applicable NFRs:**
  - NFR-003：媒體儲存冗餘（備份）支援 uptime SLA。
  - NFR-002：儲存檔案路徑而非檔案內容於 DB；存取透過 Auth middleware 控制。
- **Depends on:** Auth（authenticated upload）, Posts（媒體關聯 endpoint）
- **Key decisions:**
  - 上傳驗證：MIME type 檢查（magic bytes）+ extension 檢查 + size ≤10MB。無效時以 400 拒絕（FR-003）。
  - 縮圖：上傳時同步產生，使用 sharp/resize 至最大 320px 尺寸。
  - **版本控制（PRD「媒體資源管理與版本控制」）：** 每次上傳建立新的 Media 記錄（含唯一 filenameServer）；建立 `media_versions` 集合（或 Payload `versions.drafts`）記錄同一資源的多版本歷史。設計見 §7。
  - 儲存：local filesystem 位於 `uploads/`，以 hashed directory 結構（`uploads/ab/cd/...`）避免單一目錄 inode 限制。（若需擴展至單一伺服器以外則 TBD cloud storage。）
  - Filename：UUID + 原始 extension，避免碰撞及暴露原始檔名。

### 6.5 Categories（分類）
- **Responsibility:** 分類巢狀結構管理。
- **Serves:** FR-001, UF-002
- **Interfaces:**
  - Inbound: Payload 內建 GraphQL（Admin 後台）、`/api/v1/categories`
  - Outbound: Posts（分類關聯）
- **Data it owns / touches:** Categories（含 _children、_level）
- **Applicable NFRs:**
  - NFR-004：巢狀查詢透過 nested docs plugin 的 `_level` index 優化。
- **Depends on:** nestedDocsPlugin（巢狀支援）
- **Key decisions:**
  - 使用 Payload `nestedDocsPlugin` 支援巢狀分類（「News > Technology」）。
  - 標籤（tags）簡化為 posts 上的 `tagIds` array of text（PRD FR-001「標籤」），不建立獨立 Tags 集合以降低複雜度。

### 6.6 Content Intelligence（內容智慧 + AI）
- **Responsibility:** SEO metadata 產生、AI 輔助寫作、語氣調整、AI Provider Adapter。
- **Serves:** FR-006, FR-007, FR-008
- **Interfaces:**
  - Inbound: `/api/v1/articles/:id/seo/generate`、`/api/v1/articles/:id/ai/suggest`、`/api/v1/articles/:id/tone/adjust`
  - Outbound: External AI service（provider TBD —— FR-007/008 取決於此）、AI Audit（審批記錄）、AI Generation Records（生成記錄）
- **Data it owns / touches:** Posts（讀取內容、寫入 SEO 欄位）、AI_Generation_Records（寫入）、SEO_History（寫入）
- **Applicable NFRs:**
  - NFR-001：AI 建議回應 <10s（FR-007 驗收標準）。外部 AI 呼叫外層包覆 timeout。
  - NFR-004：外部 AI 呼叫為 stateless；每使用者 rate limiting 防止濫用。
- **Depends on:** Auth（使用者必須登入）, Posts（文章內容擷取）, AI Audit（審批記錄）
- **Key decisions:**
  - **不要拆為 microservice。** Content Intelligence 在文章編輯期間被呼叫；network hop 到獨立服務會增加延遲到已對延遲敏感的用戶流程。
  - 外部 AI 呼叫外層包覆 timeout（建議 10s、語氣調整 5s），搭配 graceful degradation —— 若 AI 不可用，回傳 `408 Request Timeout` 並提示重試。
  - SEO metadata：若使用者已手動設定任何 SEO 欄位（title/description/keywords），該欄位的自動產生即跳過（FR-006：「保留其手動設定」）。
  - **AI Provider Adapter（關鍵）：** 單一 `IAiProvider` 介面搭配具體 provider 實現。可替換而不需更動 business logic。每次呼叫寫入 AI_Generation_Records（FR-006/007/008 的生成記錄）。

### 6.7 AI Audit（審批與生成記錄）
- **Responsibility:** 審批流記錄、AI 生成記錄、SEO 歷史。
- **Serves:** FR-002, FR-006, NFR-002
- **Interfaces:**
  - Inbound: Posts 審批 endpoints（審批記錄）、Content Intelligence（生成記錄）
  - Outbound: 無直接外部消費者（audit trail）
- **Data it owns / touches:** AI_Audit、AI_Generation_Records、SEO_History
- **Applicable NFRs:**
  - NFR-002：審批與 AI 呼叫皆寫入不可篡改的 audit trail（PRD §8 限制）。
- **Depends on:** Posts（審批狀態）, Content Intelligence（生成記錄）
- **Key decisions:**
  - 每次 `reviewStatus` 變更為 AI_Audit 寫入一筆記錄（含 byUser、action、reason）。
  - 每次 AI 呼叫為 AI_Generation_Records 寫入一筆記錄（含 type、prompt、response、provider、durationMs）。
  - 每次 SEO 生成為 SEO_History 寫入一筆記錄。

### 6.8 Redirects（URL 重導向）
- **Responsibility:** URL 重導向管理。
- **Serves:** enabling（網站面）, NFR-001
- **Interfaces:**
  - Inbound: Payload 內建 redirectsPlugin（Admin 後台）、`/api/v1/redirects`
  - Outbound: Next.js frontend（重導向處理）
- **Data it owns / touches:** redirects 集合（外掛）
- **Applicable NFRs:**
  - NFR-001：重導向快取透過 on-demand revalidation 即時更新（見 §6.10）。
- **Depends on:** redirectsPlugin（外掛）
- **Key decisions:**
  - 使用 Payload `redirectsPlugin` 管理 URL 重導向（PRD Redirects）。

### 6.9 Search（全文搜尋）
- **Responsibility:** 文章全文搜尋。
- **Serves:** enabling（網站面）, NFR-001
- **Interfaces:**
  - Inbound: Payload 內建 searchPlugin（Admin 後台）、`/api/v1/search`
  - Outbound: Next.js frontend（搜尋頁面）
- **Data it owns / touches:** search 集合（外掛）
- **Applicable NFRs:**
  - NFR-001：搜尋索引透過 searchPlugin 的 beforeSync 即時更新（見 §6.10）。
- **Depends on:** searchPlugin（外掛）
- **Key decisions:**
  - 使用 Payload `searchPlugin` 對 posts 集合做全文搜尋（PRD Search）。

### 6.10 Revalidation & Jobs（On-demand Revalidation + Scheduled Publishing）
- **Responsibility:** On-demand revalidation、Scheduled Publishing、Jobs 佇列。
- **Serves:** enabling（網站面）, NFR-001, NFR-003
- **Interfaces:**
  - Inbound: Posts/Pages 集合 `afterChange` hook（觸發 revalidation）
  - Outbound: Next.js `revalidatePath`/`revalidateTag`（前端快取無刷新更新）
- **Data it owns / touches:** 無（透過 hooks 觸發）
- **Applicable NFRs:**
  - NFR-001：文章/頁面變更時透過 on-demand revalidation 即時更新前端快取（見模板 revalidatePost/revalidatePage hooks）。
  - NFR-003：Scheduled Publishing 透過 Payload Jobs 佇列在預定時間自動發布（見模板 schedulePublish）。
- **Depends on:** Next.js revalidation（前端）, Payload Jobs（排程）
- **Key decisions:**
  - 使用 Payload `versions.drafts.schedulePublish` 支援 Scheduled Publishing（PRD Jobs & Scheduled Publishing）。
  - 使用 Payload Jobs 佇列的 cron 排程（`jobs` collection）處理定時任務（見模板 schedulePublish）。
  - On-demand revalidation 透過 `afterChange` hook 呼叫 Next.js `revalidatePath`/`revalidateTag`（見模板 hooks）。

### 6.11 Layout Builder（頁面區塊編排器）
- **Responsibility:** 頁面區塊編排器。
- **Serves:** enabling（網站面）
- **Interfaces:**
  - Inbound: Payload `blocks` 類型（Pages 集合）
  - Outbound: Next.js frontend（區塊渲染）
- **Data it owns / touches:** Pages（layout blocks）
- **Applicable NFRs:**
  - NFR-001：區塊透過 SSG + revalidation 預先渲染（見 §6.3）。
- **Depends on:** Pages 集合（layout blocks）
- **Key decisions:**
  - 使用 Payload `blocks` 類型，包含 CallToAction、Content、MediaBlock、Archive、FormBlock 區塊（模板內建）。

### 6.12 Frontend SPA（網頁介面）
- **Responsibility:** 管理後台 UI + 網站前端（Next.js SSR）。
- **Serves:** 所有 FR（presentation layer）
- **Interfaces:**
  - Outbound: Payload GraphQL（Admin 後台）、自有 REST API（`/api/v1/*`，前端 SPA）、Payload REST（`/api/*`，Next.js SSR）
  - Inbound: 瀏覽器 HTTP 請求（static asset 由 backend 或 CDN 提供）
- **Data it owns / touches:** 無（stateless，透過 API 通訊）
- **Applicable NFRs:**
  - NFR-001：文章編輯器頁面 <2s 載入 —— 透過頁面載入時 initial data prefetch（articles、categories、tags、media list）達成，不使用 lazy-loading。
  - NFR-003：SPA 從 static assets 提供；backend 僅處理動態 API 呼叫。
- **Depends on:** Backend API（所有資料存取）
- **Key decisions:**
  - Admin 後台：Payload 內建 Admin UI（`/admin`），提供完整的 CMS 管理界面。
  - 網站前端：Next.js App Router（SSR/SSG），搭配 on-demand revalidation（見 §6.10）。
  - Authentication：JWT 以 httpOnly cookie 儲存（非 localStorage）以防 XSS token 竊取。
  - Editor：Lexical rich-text editor（Payload 內建）搭配內嵌 AI suggestion panel（FR-007, FR-008）。

## 7. Technology Decisions

| 決策 | 選擇 | 理由 | 拒絕的替代方案 |
|---|---|---|---|
| 核心框架 | Payload 3.x（自訂 monolith）| 已內建我們在 PRD 中列出的絕大多數後端能力：認證、RBAC、GraphQL/REST API、文檔版本與草稿、內容智慧外掛（SEO/Search/Redirects）、Jobs 佇列、Layout Builder。重寫這些會浪費大量資源且引入 bug。| 自訂 monolith（舊 v0.1，Node.js+Express）—— 需重寫認證、版本、外掛等大量既有能力，重複造輪子。|
| 資料庫 | PostgreSQL（Payload SQL adapter）| ACID compliance 用於 RBAC 和文章狀態轉換；JSONB 用於 SEO keywords；成熟生態系。| MongoDB（Payload 預設）—— PRD v0.1 選擇 PostgreSQL 用於 ACID 保證；我們保留此決策。SQLite —— 500 同時使用者不足（NFR-001）。|
| Session / Auth | JWT（access）+ refresh token（httpOnly cookie）| Stateless access tokens 用於 API；cookie 中的 refresh tokens 防止 XSS token 竊取。| Server-side sessions —— 此規模下不必要的 state；localStorage tokens —— XSS vulnerable。|
| Password hashing | bcrypt（cost factor 12）| FR-004 要求 salted hashing；bcrypt 經得起考驗且符合 NFR-002。| Argon2id —— 安全性更好但 500 同時登入時 CPU 成本較高；PBKDF2 —— 對 GPU 攻擊較弱。|
| File storage | Local filesystem（uploads/ with hashed directories）| 簡單、快速，適合 10K DAU 單一伺服器部署。| S3/cloud storage —— 目前規模過頭；增加營運複雜度。|
| Thumbnail generation | sharp（Node.js 影像處理）| 快速、compiled、不依賴系統 library。| GraphicsMagick/ImageMagick —— 較重的系統依賴。|
| AI integration | Adapter pattern with pluggable provider | FR-007/008 指定「TBD provider」；adapter 允許無程式碼變更下替換。| Hard-coded provider call —— 在選擇 provider 前就鎖定。|
| 自有 API | 自訂版本化 REST（`/api/v1/*`）+ Payload GraphQL（後台）| Foan 要求「保留自有 API」；自訂 REST 提供顯式、文檔化的 endpoint 回應 PRD FR；GraphQL 供後台彈性查詢。| 只用 Payload GraphQL —— 無显式 REST API 回應 PRD FR；只用 Payload REST —— 無版本化與 PRD 對應。|
| Frontend | Next.js App Router + Payload Admin | Next.js 支援 SSR/SSG 滿足 NFR-001；Payload Admin 提供完整 CMS 管理界面。| Server-rendered HTML templates —— 較難維護互動式 editor state。|

## 8. 非功能需求如何達成

| NFR | 設計如何滿足 |
|---|---|
| NFR-001（效能：編輯器頁面 <2s，500 同時使用者，p95）| 單一 monolith 消除跨服務延遲。文章編輯器資料以單一 API 呼叫預先擷取。PostgreSQL on `_status`、`categoryId`、`reviewStatus` 建 index。SSR 用於初始頁面載入。On-demand revalidation（§6.10）確保前端快取即時更新。若基準測試顯示 p95 >2s，可選配 Redis 快取經常存取的文章資料。|
| NFR-002（安全：密碼雜湊、PII 加密）| 所有密碼使用 bcrypt cost 12。所有流量使用 TLS。密碼雜湊僅以 salted values 儲存。JWT refresh tokens 在 httpOnly cookies。PII（email、reset tokens）以 PostgreSQL pgcrypto 或 application-level encryption 加密 at rest。GDPR 合規：資料匯出/刪除 endpoints、同意紀錄。審流與 AI 呼叫皆寫入 audit trail（AI_Audit、AI_Generation_Records）。|
| NFR-003（可用性：營運時間內 99.9% uptime）| Monolith 部署於單一 container 搭配 health check。上傳檔案備份。資料庫每日備份。Graceful degradation：若 AI service 當機，編輯器仍正常運作（SEO/手動內容）。生產前在 500 同時使用者下進行 load testing。|
| NFR-004（延展性：10,000 DAU）| PostgreSQL connection pooling（必要時 PgBouncer）。靜態媒體可透過 CDN 提供（選用）。Stateless API 設計允許水平擴展 backend（如需）。無需 session affinity（JWT-based auth）。|

## 9. Risks & Trade-offs

- **風險：** 外部 AI 服務提供者尚未選擇（PRD §10 開放問題）—— 阻擋 Content Intelligence 實作。**緩解：** Adapter 介面設計為 provider-agnostic；開發期間可使用 mock 實現。
- **風險：** 自訂 monolith（Payload）的複雜度高於自訂 Express monolith。**緩解：** Payload 已內建我們所需的絕大多數能力；自訂集中在 hooks/access/plugins，而非重寫核心。
- **風險：** PostgreSQL schema 變更可能遺失數據（Payload SQL adapter 特性）。**緩解：** 開發環境使用 `push: true`；生產環境使用 migration 並謹慎評估 schema 變更影響。
- **風險：** 若範圍成長，Monolith 可能難以維護。**緩解：** Clean 分層架構（collections → hooks → access → plugins）強制邊界，使未來分解更容易。
- **風險：** GDPR 合規需要資料匯出/刪除 —— PRD 的 FR 中未明確列出。**緩解：** 設計 Users entity 含 soft-delete 和匯出能力；GDPR 是硬限制（PRD §8），即使未列為 FR 也必須實作。

## 10. 開放問題

- **回答者：** Planner / Foan
  - 具體的外部 AI 服務提供者為何？（PRD §10：「specific provider TBD」）—— 影響 adapter 實作細節。
- **回答者：** Planner / Foan
  - 除 GDPR 外的區域資料安全要求為何？（PRD §10：GDPR/CCPA）—— 影響加密範圍和資料 residency 決策。
- **回答者：** Architect（自行解決）
  - Frontend SSR framework 選擇（Next.js vs. custom SSR）—— 採用 Next.js（Payload 模板內建，滿足 NFR-001）。
- **回答者：** Architect（自行解決）
  - 媒體版本控制的具體實作（独立 media_versions 集合 vs. Payload versions.drafts）—— 採用独立 media_versions 集合（見 §5.2），因 PRD 要求「版本控制」指資源的多版本歷史，而非草稿。

---

_Gate 2 核准授權建立 accompanying **Task Breakdown** 中定義的實作任務。本文與 Task Breakdown 一併核准。_
