# 架構設計：OU Know v3（兩獨立 Repo：前端 presentation + 後端 API）

> **撰寫規範（Architect 必須遵守）：**
> 1. 本文回答 PRD 未開的 **how**。不重新開啟或重新協商需求 —— 若需求有誤或不清楚，提出澄清，不要靜默重新設計。
> 2. 每個設計決策都追溯至需求（FR-XXX / NFR-XXX）。若 component 未對應到任何需求，以 enabling 工作說明或刪除此 component。
> 3. 明確定義 component 之間的 contract —— 特別是 Dev-be / Dev-fe 邊界，以及本 v3 的 **Repo A ↔ Repo B 邊界**。兩個 agent 對著模糊介面開發，會做出不相容的兩半。
> 4. 說明 trade-off 與 rejected alternatives。沒有 trade-off 的設計是沒有思考過的設計。
> 5. 不在本文分配或排程工作 —— 那是 Task Breakdown 的事。本文是「它長什麼樣子」，不是「誰做什麼、何時做」。
> _(提交前刪此地塊。)_

## Meta

| 欄位 | 值 |
|---|---|
| ProjectId | PJ-0002 |
| 作者 | Architect |
| 日期 | 2026-09-06 |
| 版本 | 0.3 |
| 基於 PRD 版本 | 0.1 |
| Gate 2 核准者 | Foan / Mike |
| 狀態 | Draft |
| 取代 | v0.2（commit 7b0d2d6，單 monolith 同程序）|

---

## 1. Approach

OU Know v3 是一個統一的數位內容管理平台，整合兩個來源的功能面：`ouknow_website`（Payload 3.x Website Template）的完整 CMS 能力（Posts、Pages、Media、Categories、Projects、SEO、Search、Redirects、Draft/Live Preview、On-demand Revalidation、Jobs & Scheduled Publishing、Layout Builder、Auth/Access Control），以及原 PJ-0002 PRD 的內容管理與 AI 驅動功能（文章 CRUD + 狀態流、分類標籤、媒體版本控制、帳號與 RBAC、SEO metadata 生成、AI 輔助寫作、語氣調整）。

解決方案為**兩個獨立部署單元的前後端分離**：**Repo A（前端）** = `ouknow_website` 模板的 Next.js presentation（網站 SSR + Admin 後台 UI），純展示、不連 DB，透過 HTTP call Repo B；**Repo B（後端）** = 獨立 API 服務，負責 PostgreSQL、Auth/RBAC、自有 REST API、AI adapter、Jobs，不渲染 HTML。兩者之間只走 HTTP API 契約（§4），不是進程內 call。最重要的設計決策是**將 Payload 的 backend（collections/auth/jobs/DB）以 headless 模式透過 `payload.handler()` 在獨立 HTTP server 上提供**——這既重用了模板後端的全部能力，又滿足「前端無後端邏輯、後端獨立可部署」的硬要求。

## 2. 架構風格與規範

**風格：前後端分離的兩個 monolith（已論證）**

| 面向 | 決策 | 為何適合此專案規模 |
|---|---|---|
| 架構風格 | 兩個 monolith 前後端分離（Repo A 純 presentation monolith；Repo B 純 API monolith）| 功能面廣（兩個來源整合）但每個單元仍是一個清晰的關注點：一個負責展示，一個負責資料。無獨立部署服務的需求到需要各自獨立 scale 的程度。HTTP API 契約讓兩者可分別部署、各自扩容。過度拆成微服務會引入分散式交易與網路延遲 overhead；維持兩個 monolith 是此規模的最佳平衡。 |

**所有程式碼必須遵守的規範：**
- **Clean code 基線（不可妥協）：** 清晰的命名、單一職責函式、無死碼或重複程式碼、可讀的結構。Code only a machine can maintain is a defect。
- **Repo 邊界（不可逾越）：** Repo A **不得**匯入、執行或 build Payload 的 `buildConfig`、collections、plugins、DB adapter 或任何後端設定；Repo A 只透過 HTTP client 呼叫 Repo B。Repo B **不得**渲染 HTML（不產生 `.next` 产物、不用 `next build`）；Repo B 只回傳 JSON（REST/GraphQL）。違反此邊界即 defect。
- **API 契約優先：** Repo A 與 Repo B 之間的所有協定寫在 §4 的契約表。任何 endpoint 的增刪必須同步更新契約表與兩端程式碼。
- **程式碼組織（Repo B）：** `src/collections/*`（資料集合設定）→ `src/collections/<Name>/hooks/*`（集合級 hooks：beforeChange/afterChange/afterRead，強制狀態機）→ `src/access/*`（跨集合存取控制）→ `src/plugins/*`（SEO/Search/Redirects 外掛）→ `src/adapters/*`（AI Provider Adapter）→ `src/routes/*`（自訂 `/api/v1/*` REST endpoints）→ `src/utilities/*`。禁止在 server 入口直接寫 business logic。
- **程式碼組織（Repo A）：** `src/app/*`（Next.js App Router 頁面）→ `src/components/*`（UI 元件）→ `src/lib/api/*`（對 Repo B 的唯一 HTTP client 封裝）→ `src/types/*`（由 API contract/generated types 產生）。禁止任何元件直接碰 DB 或匯入 Repo B 設定。
- **命名規範：** collection slug 用 kebab-case；field 名用 camelCase；REST endpoint 路徑 `/api/v1/<resource>`（版本化）。
- **錯誤處理：** 所有錯誤遵循結構化 `ApiError` 格式 `{code, message, details}`。Repo B 的 raw exception 不得洩漏至 HTTP response；Repo A 的 HTTP client 統一轉換 Repo B 錯誤碼為前端錯誤顯示。

## 3. Component Map _(global view)_

### 3.1 Repo A（前端 presentation）

| Component | 職責 | 服務於 | 詳細 |
|---|---|---|---|
| Website（Next.js App Router）| 網站前端 SSR/SSG/SPA | 所有 FR（presentation）, NFR-001 | §6.1 |
| Admin UI | CMS 管理後台介面 | FR-001, FR-002, FR-003, FR-005 | §6.2 |
| API Client（對 Repo B）| 對 Repo B 的唯一 HTTP 封裝 | 所有資料存取 | §6.3 |

### 3.2 Repo B（後端 API）

| Component | 職責 | 服務於 | 詳細 |
|---|---|---|---|
| Payload API Server | 以 headless 模式提供 GraphQL/REST API、DB、Auth | FR-003, FR-004, FR-005, NFR-002 | §6.4 |
| Posts | 文章 CRUD、PJ-0002 狀態流、審流、分類、標籤、作者 | FR-001, FR-002, UF-001 | §6.5 |
| Pages | 靜態頁面、Layout Builder、SEO | enabling（網站面）, NFR-001 | §6.6 |
| Media | 上傳驗證、儲存、縮圖、版本控制 | FR-003, UF-002, NFR-003 | §6.7 |
| Categories | 分類巢狀結構 | FR-001, UF-002 | §6.8 |
| Content Intelligence（AI）| SEO metadata 生成、AI 輔助寫作、語氣調整、AI Adapter | FR-006, FR-007, FR-008 | §6.9 |
| AI Audit（審批與生成記錄）| 審流記錄、AI 生成記錄、SEO 歷史 | FR-002, FR-006, NFR-002 | §6.10 |
| Redirects | URL 重導向 | enabling（網站面）, NFR-001 | §6.11 |
| Search | 文章全文搜尋 | enabling（網站面）, NFR-001 | §6.12 |
| Jobs & Revalidation | Scheduled Publishing、On-demand Revalidation、Jobs 佇列 | enabling, NFR-001, NFR-003 | §6.13 |

_（兩 Repo 關係圖見 §3.3）_

### 3.3 關係圖

```
┌───────────────────────── Repo A（前端 presentation，不連 DB）─────────────────────────┐
│                                                                                      │
│  ┌────────────────────────┐              ┌────────────────────────┐                   │
│  │   Website（Next.js）    │              │    Admin UI（前端）      │                   │
│  │  網站 SSR/SSG/SPA       │              │  CMS 管理後台介面        │                   │
│  └────────────┬───────────┘              └────────────┬───────────┘                   │
│               │                                        │                              │
│               └──────────────┬─────────────────────────┘                              │
│                              ▼                                                       │
│                    ┌──────────────────┐                                              │
│                    │  API Client       │  唯一 HTTP 封裝（不碰 DB、不匯入後端設定）      │
│                    └────────┬─────────┘                                              │
└──────────────────────────────┼───────────────────────────────────────────────────────┘
                               │  HTTP（REST + GraphQL，JWT，HTTPS）— §4 契約
                               ▼
┌───────────────────────── Repo B（後端 API，不渲染 HTML）──────────────────────────────┐
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │  Payload API Server（payload.handler() on standalone HTTP server）     │           │
│  │  ┌────────────┐ ┌───────────┐ ┌────────────┐ ┌────────────────────┐  │           │
│  │  │ /api/v1/*  │ │ GraphQL   │ │ Auth/RBAC  │ │ AI Adapter (IAiProv) │  │           │
│  │  └─────┬──────┘ └─────┬─────┘ └─────┬──────┘ └─────────┬──────────┘  │           │
│  └────────┼──────────────┼─────────────┼──────────────────┼───────────┘           │
│           │              │              │               │                          │
│  ┌────────▼──────────────▼──────────────▼───────────────▼─────────────────────────┐│
│  │  Payload backend core（collections / hooks / access / plugins / jobs）          ││
│  │  Posts · Pages · Media · Categories · Users · AI_Audit · AI_Gen_Records · SEO_H ││
│  └──────────────────────────┬─────────────────────────────────────────────────────┘│
│                             ▼                                                       │
│                    ┌──────────────────┐                                             │
│                    │  PostgreSQL       │  所有資料表                                  │
│                    └──────────────────┘                                             │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## 4. Repo A ↔ Repo B API 契約 _(本 v3 的關鍵 seam)_

> 這是 v3 最重要的設計。Repo A 的所有資料存取必須經過 §6.3 的 API Client，且只接受 §4 契約中的 endpoint。任何不在契約中的呼叫都是 violation。

### 4.1 傳輸與安全

| 項目 | 契約 |
|---|---|
| 傳輸協議 | HTTPS ONLY。Repo A 的所有 HTTP client 呼叫必須走 `https://`。 |
| 認證 | Bearer JWT（`Authorization: Bearer <jwt>`）。Access token TTL 15 min；Refresh token 以 httpOnly cookie 存在 Repo A 端，由 Repo A 的 API Client 在快取過期時自動 refresh。 |
| 版本化 | 所有 endpoint 路徑前綴 `/api/v1/`。Future major version 用 `/api/v2/`。 |
| 內容類型 | `Content-Type: application/json`（multipart upload 除外）。 |
| 時區 | 所有 timestamp 欄位用 ISO 8601 UTC。 |
| 錯誤格式 | 一律回傳 `{ "code": string, "message": string, "details": any }`。HTTP status 碼見各 endpoint。Repo A 的 API Client 統一將 Repo B 的 `code` 轉為前端錯誤提示。 |
| Rate limit | Repo B 對外部消費者 endpoint 實施 per-user rate limit（防濫用，NFR-004）。超限時回傳 `429` + `Retry-After` header。 |
| CORS | Repo B 的 CORS whitelist 為 Repo A 的 production domain（`getServerSideURL()`）。 |

### 4.2 自有 REST API（Repo B 提供，Repo A 呼叫）

| From → To | Contract（endpoint / request / response） | HTTP status | Notes |
|---|---|---|---|
| A → B | `POST /api/v1/auth/register` `{email, password, name, role?}` → `{user, token}` | 201 | password ≥8；role 預設 author（FR-005）|
| A → B | `POST /api/v1/auth/login` `{email, password}` → `{user, token}` | 200 / 401 | 5 次失敗 lockout（FR-003）|
| A → B | `POST /api/v1/auth/logout` → `{}` | 204 | 清除 cookie |
| A → B | `POST /api/v1/auth/refresh` `{refreshToken}` → `{token}` | 200 / 401 | API Client 自動呼叫 |
| A → B | `POST /api/v1/auth/password-reset/request` `{email}` → `{}` | 204 | email 發 reset link（FR-003）|
| A → B | `POST /api/v1/auth/password-reset/confirm` `{token, newPassword}` → `{}` | 204 / 400 | token 有效 1 小時，單次使用 |
| A → B | `GET /api/v1/articles[/:id]` | 200 / 404 | published 可見；draft 需 auth（FR-001）|
| A → B | `POST /api/v1/articles` `{title, content, categoryId?, tagIds[], status?}` → `{article}` | 201 | 標題必填（FR-001）|
| A → B | `PUT /api/v1/articles/:id` `{partial}` → `{article}` | 200 / 403 | |
| A → B | `DELETE /api/v1/articles/:id` → `{}` | 204 / 403 | 僅 admin/editor |
| A → B | `POST /api/v1/articles/:id/to-review` → `{article}` | 200 | draft → review（FR-002 審流）|
| A → B | `POST /api/v1/articles/:id/approve` → `{article}` | 200 | review → published（FR-002）|
| A → B | `POST /api/v1/articles/:id/reject` `{reason}` → `{article}` | 200 | review → draft + 記錄原因（FR-002）|
| A → B | `POST /api/v1/articles/:id/publish` → `{article}` | 200 | draft → published |
| A → B | `POST /api/v1/articles/:id/archive` → `{article}` | 200 / 403 | 僅 admin |
| A → B | `POST /api/v1/articles/:id/media` `{mediaIds[]}` → `{article}` | 200 | 關聯媒體（UF-002）|
| A → B | `POST /api/v1/media/upload` (multipart/form-data) → `{media, thumbnailUrl}` | 201 / 400 | ≤10MB，類型驗證（FR-003）|
| A → B | `GET /api/v1/media[/:id]` | 200 / 404 | |
| A → B | `POST /api/v1/articles/:id/seo/generate` → `{seo}` | 200 / 408 | 自動 SEO（FR-006）|
| A → B | `POST /api/v1/articles/:id/ai/suggest` `{paragraph?, focus?}` → `{suggestions[]}` | 200 / 408 | FR-007（<10s）|
| A → B | `POST /api/v1/articles/:id/tone/adjust` `{paragraph, tone}` → `{rewritten}` | 200 / 408 | FR-008 |
| A → B | `GET /api/v1/categories` → `{categories}` | 200 | 含巢狀 |
| A → B | `GET /api/v1/search?q=...&limit=&offset=` → `{results}` | 200 | 全文搜尋 |
| A → B | `GET /api/v1/redirects` → `{redirects}` | 200 | |
| A → B | `GET /api/v1/users`（admin）→ `{users}` | 200 / 403 | 使用者管理（FR-005）|
| A → B | `POST /api/v1/users` `{email, password, name, role}` → `{user}` | 201 / 400 | |
| A → B | `PUT /api/v1/users/:id` `{partial}` → `{user}` | 200 | |
| A → B | `DELETE /api/v1/users/:id` → `{}` | 204 / 403 | |

_所有 endpoint 皆需 JWT（除 login/register/password-reset）。_

### 4.3 GraphQL（Repo B 提供，Repo A 的 Admin UI 使用）

| 項目 | 契約 |
|---|---|
| Endpoint | `POST /graphql`（Repo B）|
| 使用方 | Repo A 的 Admin UI（CRUD、filter、populate）|
| Auth | 同 4.1（Bearer JWT）|
| Schema | Payload 內建 GraphQL schema（由 Repo B 的 collections 產生）|

### 4.4 狀態轉換事件（Repo B 內部 event，不跨 Repo）

| 事件 | 觸發 | 消費 |
|---|---|---|
| `posts._status` 變更 | Posts `beforeChange` hook | revalidation hook、AI Audit 記錄 |
| `posts.reviewStatus` 變更 | 審批 endpoints | AI Audit 記錄 |
| `ai_generation_record` 建立 | AI service | audit trail |

## 5. Data Model _(Repo B 獨有；Repo A 不儲存任何資料)_

### 5.1 核心集合（Collections）

#### Users（認證與 RBAC）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key（Postgres）|
| email | Unique text | 登入憑證 |
| password | Hashed | bcrypt/argon2 salted hash（Payload auth）|
| name | text | 顯示名稱 |
| role | select(`admin`/`editor`/`author`) | RBAC（FR-005）；**新增** |
| resetToken | text | NULL = 無待處理 reset |
| resetTokenExpiry | date | NULL = 無待處理 reset |
| loginAttempts | number | 成功登入後重設 0（FR-003）|
| lockedUntil | date | NULL = 未 lock |
| createdAt / updatedAt | date | Payload timestamps |

#### Posts（文章 + PJ-0002 狀態流）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| title | text（required）| 標題 |
| content | richText（Lexical，required）| 內容（含 blocks）|
| slug | text（unique）| URL-friendly |
| `_status` | select(`draft`/`published`/`archived`) | Payload drafts `_status`；**新增** `archived`（FR-002）|
| `reviewStatus` | select(`pending`/`approved`/`rejected`) | **新增**：審流（FR-002）|
| `reviewReason` | text | 拒絕原因 |
| `reviewedBy` | relationship → users | 審批者 |
| `reviewedAt` | date | 審批時間 |
| categoryId | relationship → categories | 單一分類 |
| tagIds | array of text | 標籤（簡化多對多）|
| authors | relationship → users (many) | 作者 |
| meta.title / meta.description / meta.image | group（SEO）| SEO（FR-006）|
| relatedPosts | relationship → posts (many) | 相關文章 |
| heroImage | upload → media | 頭圖 |
| publishedAt | date | 發表時間 |

> **狀態機轉譯：** PRD draft/review/published/archived = Payload `_status`（draft/published/archived）+ 新增 `reviewStatus`（pending/approved/rejected）。`reviewStatus` 在 draft 與 published 之間作為審批門檻。

#### Pages（靜態頁面）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| title | text | 頁面標題 |
| slug | text | URL-friendly（home = 首頁）|
| hero | group（type / richText / links / media）| Hero |
| layout | blocks（CallToAction/Content/MediaBlock/Archive/FormBlock）| Layout Builder |
| meta | group（SEO）| SEO |
| publishedAt | date | 發表時間 |
| `_status` | select | Payload drafts `_status` |

#### Media（數位資源 + 版本控制）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| filenameServer | text | 系統產生，唯一 |
| mimetype | text | 上傳驗證 |
| size | number | 位元組 |
| url | text | 存取 URL |
| thumbnailUrl | text | 縮圖 URL |
| uploadedBy | relationship → users | 上傳者 |
| createdAt | date | 建立時間 |

> **版本控制：** 每次上傳建立新的 Media 記錄（唯一 filenameServer）；另建 `media_versions` 集合記錄同一資源多版本歷史（見 §7）。

#### Categories（分類）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| name | text | 分類名稱 |
| slug | text | URL-friendly |
| _children | relationship → categories | 巢狀子分類 |
| _level | number | 巢狀層級 |

### 5.2 外掛集合

| 集合 | 來源 | 備註 |
|---|---|---|
| redirects | redirectsPlugin | URL 重導向 |
| search | searchPlugin | 文章全文搜尋索引 |
| forms, formSubmissions | formBuilderPlugin | 表單與回應 |

### 5.3 AI 與審批專屬集合（PRD 額外需求）

#### AI_Audit（審批與稽核記錄）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| action | select(`create`/`to_review`/`approve`/`reject`/`publish`/`archive`) | 審批動作 |
| byUser | relationship → users | 執行者 |
| reason | text | 備註/拒絕原因 |
| createdAt | date | 審批時間 |

> 對應 FR-002（審流）與 NFR-002（稽核日誌）。

#### AI_Generation_Records（AI 生成記錄）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| type | select(`seo`/`suggest`/`tone`) | 生成類型 |
| prompt | text | 原始輸入 |
| response | text | 生成結果 |
| provider | text | 使用的 provider |
| tokenUsage | number | 使用 token 數 |
| durationMs | number | 處理時間（FR-007：<10s）|
| createdAt | date | 生成時間 |

#### SEO_History（SEO 元資料歷史）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| articleId | relationship → posts | 關聯文章 |
| title / description | text | 產生的 meta |
| keywords | json | 產生的 keywords |
| generatedBy | select(`ai`/`manual`) | 來源 |
| createdAt | date | 產生時間 |

#### media_versions（媒體版本控制）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| mediaId | relationship → media | 關聯原資源 |
| version | number | 版本號 |
| filenameServer | text | 該版本檔案 |
| uploadedBy | relationship → users | 上傳者 |
| createdAt | date | 建立時間 |

## 6. Component Detail _(per-component — the unit a developer reads)_

### 6.1 Website（Repo A — Next.js App Router）
- **Responsibility:** 網站前端 SSR/SSG/SPA，純 presentation。
- **Serves:** 所有 FR（presentation layer）, NFR-001
- **Interfaces:**
  - Outbound: 透過 API Client（§6.3）呼叫 Repo B 的 REST（`/api/v1/*`）與 GraphQL（`/graphql`）。
  - Inbound: 瀏覽器 HTTP 請求（static asset）。
- **Data it owns / touches:** 無（stateless，不連 DB）。
- **Applicable NFRs:**
  - NFR-001：網站載入透過 Next.js SSG + client-side revalidation 達成。
- **Depends on:** Repo B API（所有資料存取）。
- **Key decisions:**
  - 移除此模板的 `withPayload` 與在地的 `buildConfig` 匯入（v3 關鍵變更）—— Repo A 不再 run 後端。
  - SSR/SSG 資料擷取全部走 Repo B 的 HTTP API。
  - Draft/Live Preview：透過 HTTP 呼叫 Repo B 的 preview endpoint 取得 draft 內容後渲染（見 §7 的 preview 重設計）。

### 6.2 Admin UI（Repo A — 前端）
- **Responsibility:** CMS 管理後台介面。
- **Serves:** FR-001, FR-002, FR-003, FR-005
- **Interfaces:**
  - Outbound: 透過 API Client 呼叫 Repo B 的 GraphQL（`/graphql`）與 REST（`/api/v1/*`）。
- **Data it owns / touches:** 無。
- **Applicable NFRs:**
  - NFR-001：Admin UI 載入透過 code splitting + 對 Repo B 的前端快取。
- **Depends on:** Repo B API。
- **Key decisions:**
  - **見 §7 的 Open Question：Admin UI 的選擇（lightweight custom vs. Payload stock admin）——這是本 v3 的關鍵架構抉擇。**

### 6.3 API Client（Repo A — 對 Repo B 的唯一 HTTP 封裝）
- **Responsibility:** 封裝對 Repo B 的所有 HTTP 呼叫（REST + GraphQL），集中處理 JWT、refresh、錯誤轉換。
- **Serves:** 所有資料存取（Repo A 側）。
- **Interfaces:**
  - Outbound: Repo B 的 `/api/v1/*`、`/graphql`。
  - Inbound: Repo A 的所有元件。
- **Key decisions:**
  - Repo A 中**唯一**允許碰 network 的模組。所有其它元件不得直接 `fetch` Repo B。
  - 集中處理 access token 注入、refresh 重試、Repo B 錯誤碼 → 前端錯誤訊息轉換。

### 6.4 Payload API Server（Repo B — headless）
- **Responsibility:** 以 headless 模式提供 GraphQL/REST API、DB、Auth、Jobs。
- **Serves:** FR-003, FR-004, FR-005, NFR-002, NFR-003
- **Interfaces:**
  - Inbound: Repo A 的 HTTP 呼叫（§4）、外部消費者。
  - Outbound: PostgreSQL（DB adapter）、AI Adapter（§6.9）。
- **Data it owns / touches:** 所有 collections（§5）。
- **Applicable NFRs:**
  - NFR-002：bcrypt、TLS、PII 加密 at rest。
  - NFR-001：API 回應 <100ms p95（indexed lookup）。
  - NFR-003：API server health check、独立 deploy。
- **Depends on:** PostgreSQL、AI Adapter。
- **Key decisions:**
  - **以 `payload.handler()` 在獨立 HTTP server（Fastify 或 Express）上提供 API**——這是 Payload 官方提供的 headless 模式，讓 collections/auth/jobs/DB 獨立於 Next.js 渲染之外可部署。
  - Repo B **不執行** `next build`、不渲染 HTML、不產生 `.next` 产物。
  - DB 用 PostgreSQL（`@payloadcms/db-postgres`，取代模板預設的 sqlite）。
  - GraphQL endpoint `/graphql` 供 Repo A Admin UI 使用。

### 6.5 Posts（Repo B）
- **Responsibility:** 文章 CRUD、PJ-0002 狀態流、審流、分類、標籤、作者、SEO。
- **Serves:** FR-001, FR-002, UF-001
- **Interfaces:**
  - Inbound: `/api/v1/articles*`（§4.2）、GraphQL。
  - Outbound: Content Intelligence（SEO auto-fill）、Media、AI Audit（§6.10）。
- **Data it owns / touches:** Posts（含 `_status`、`reviewStatus`、`reviewedBy`、`reviewedAt`）、Categories、Media。
- **Applicable NFRs:**
  - NFR-001：PostgreSQL on `_status`、`categoryId`、`reviewStatus` 建 index。
  - NFR-004：posts table index 以利過濾。
- **Depends on:** Auth（RBAC）, Content Intelligence, Media, AI Audit。
- **Key decisions:**
  - 狀態機在 `beforeChange` hook 強制（FR-002）：
    - `draft` → `to-review`（author 提交）→ `reviewStatus=pending`
    - `pending` → `published`（editor approve）+ `approved`
    - `pending` → `draft`（editor reject）+ `rejected` + `reviewReason`
    - `draft` → `published`（admin 直接發表）
    - `published` → `draft`（revert；FR-002 驗收標準）
    - `draft` → `archived`（僅 admin）
  - `reviewedBy`/`reviewedAt` 在 approve/reject 設定；`publishedAt` 在 publish 設定。

### 6.6 Pages（Repo B）
- **Responsibility:** 靜態頁面、Layout Builder、SEO。
- **Serves:** enabling, NFR-001
- **Interfaces:** Inbound: GraphQL、`/api/v1/*`；Outbound: Media、SEO plugin。
- **Data:** Pages（layout blocks、hero、meta）。
- **Depends on:** Media, SEO plugin, Layout Builder。
- **Key decisions:** Layout Builder 用 Payload `blocks`（CallToAction/Content/MediaBlock/Archive/FormBlock）。

### 6.7 Media（Repo B）
- **Responsibility:** 上傳驗證、儲存、縮圖、版本控制。
- **Serves:** FR-003, UF-002, NFR-003
- **Interfaces:** Inbound: `/api/v1/media/upload`、`/api/v1/media`；Outbound: file storage、image processing。
- **Data:** Media、media_versions。
- **Applicable NFRs:** NFR-003（冗餘備份）、NFR-002（存路徑非內容）。
- **Key decisions:** MIME（magic bytes）+ extension + ≤10MB 驗證；sharp 縮圖 ≤320px；local filesystem `uploads/` hashed dirs；版本控制見 §5.2/§7。

### 6.8 Categories（Repo B）
- **Responsibility:** 分類巢狀結構。
- **Serves:** FR-001, UF-002
- **Depends on:** nestedDocsPlugin。
- **Key decisions:** nested docs 支援巢狀（「News > Technology」）。

### 6.9 Content Intelligence（AI，Repo B）
- **Responsibility:** SEO 生成、AI 寫作輔助、語氣調整、AI Adapter。
- **Serves:** FR-006, FR-007, FR-008
- **Interfaces:** Inbound: `/api/v1/articles/:id/seo/generate`、`/ai/suggest`、`/tone/adjust`；Outbound: External AI service、AI Audit、AI_Generation_Records。
- **Data:** Posts（讀）、AI_Generation_Records、SEO_History（寫）。
- **Applicable NFRs:** NFR-001（AI <10s，timeout 包覆）、NFR-004（stateless + rate limit）。
- **Depends on:** Auth, Posts, AI Audit, External AI service。
- **Key decisions:**
  - **AI Provider Adapter（IAiProvider）：** 單一介面搭配 provider-specific 實現，provider-agnostic（FR-007 TBD）。每次呼叫寫入 AI_Generation_Records。
  - 外部 AI 呼叫包覆 timeout（SEO 10s、語氣 5s）+ graceful degradation（回傳 408）。
  - SEO：使用者已手動設定之欄位跳過自動生成（FR-006）。

### 6.10 AI Audit（審批與生成記錄，Repo B）
- **Responsibility:** 審流記錄、AI 生成記錄、SEO 歷史。
- **Serves:** FR-002, FR-006, NFR-002
- **Interfaces:** Inbound: Posts 審批 endpoints、Content Intelligence；Outbound: 無外部消費者（audit trail）。
- **Data:** AI_Audit、AI_Generation_Records、SEO_History、media_versions。
- **Applicable NFRs:** NFR-002（不可篡改 audit trail）。
- **Key decisions:** 每次 `reviewStatus` 變更寫 AI_Audit；每次 AI 呼叫寫 AI_Generation_Records；每次 SEO 生成寫 SEO_History。

### 6.11 Redirects（Repo B）
- **Responsibility:** URL 重導向。
- **Serves:** enabling, NFR-001
- **Depends on:** redirectsPlugin。
- **Key decisions:** Payload redirectsPlugin。

### 6.12 Search（Repo B）
- **Responsibility:** 文章全文搜尋。
- **Serves:** enabling, NFR-001
- **Depends on:** searchPlugin。
- **Key decisions:** Payload searchPlugin on posts。

### 6.13 Jobs & Revalidation（Repo B）
- **Responsibility:** Scheduled Publishing、On-demand Revalidation、Jobs 佇列。
- **Serves:** enabling, NFR-001, NFR-003
- **Interfaces:** Inbound: Posts/Pages `afterChange` hook；Outbound: Repo A 的 revalidation webhook（`POST /api/v1/revalidate`）或 Repo A 端的 Next.js `revalidatePath`（若 Preview 在同一 domain）。
- **Data:** 無（透過 hooks 觸發）。
- **Applicable NFRs:** NFR-001（即時快取更新）、NFR-003（排程發布）。
- **Depends on:** Next.js revalidation（Repo A）、Payload Jobs（排程）。
- **Key decisions:**
  - Payload `versions.drafts.schedulePublish` + Jobs cron 排程。
  - On-demand revalidation：由於網站（Repo A）與 API（Repo B）分離，revalidation 改為 Repo B 在文檔變更時 `POST` webhook 到 Repo A 的 `/api/v1/revalidate`（或透過 shared 的 revalidation secret + Next.js `unstable_revalidate` route），而非模板的進程內 `revalidatePath`。見 §7。

## 7. Technology Decisions

| 決策 | 選擇 | 理由 | 拒絕的替代方案 |
|---|---|---|---|
| 部署結構 | 兩個獨立 monolith（Repo A presentation / Repo B API）| Foan 硬要求：前端無後端邏輯、後端獨立可部署、各自 scale。HTTP 契約讓兩者可分別部署。| 單 monolith 同程序（v2，commit 7b0d2d6）—— 违反 Foan 指示；Next.js 與後端同程序無法獨立 scale。|
| 後端框架 | Payload 3.x backend（headless，`payload.handler()` on Fastify/Express）| 模板後端已實作全部所需能力（collections、RBAC、lexical、drafts、SEO/Search/Redirects、jobs）；`payload.handler()` 是官方 headless 模式，讓 API 獨立部署。重寫這些會浪費資源且引入 bug。| 完全重寫為 Express/NestJS + Prisma—— 需重造認證、版本、外掛等大量既有能力，重複造輪子，回歸風險高。|
| Payload 耦合的取捨 | 採用 headless API + lightweight Admin（見 §6.2 / Open Question）| 保留 Payload 全部後端能力，同時滿足「前端 repo 不含後端設定」的邊界。| 使用 Payload stock Admin（需 co-located build-time config——會把後端 schema 洩漏進前端 repo，違反兩 repo 邊界）。|
| 資料庫 | PostgreSQL（`@payloadcms/db-postgres`）| ACID 用於 RBAC 與狀態轉換；PRD v2 已選 PG。| SQLite（模板預設）—— 500 同時使用者不足（NFR-001）；MongoDB—— 無 ACID 用於交易型 auth。|
| Session / Auth | JWT（access）+ refresh token（httpOnly cookie，Repo A 端）| Stateless access；cookie refresh 防 XSS。| Server-side sessions—— 此規模下 unnecessary；localStorage tokens—— XSS vulnerable。|
| Password hashing | bcrypt（cost 12）| FR-004 salted hashing；符合 NFR-002。| Argon2id—— CPU 成本較高；PBKDF2—— 對 GPU 攻擊較弱。|
| File storage | Local filesystem（Repo B，uploads/ hashed dirs）| 簡單、快速，適合 10K DAU。| S3/cloud—— 目前規模過頭。|
| Thumbnail | sharp | 快速、compiled。| ImageMagick—— 較重系統依賴。|
| AI integration | IAiProvider adapter（provider-agnostic）| FR-007/008 provider TBD；可替換。| Hard-coded provider—— 選定前就鎖定。|
| 前端 | Next.js App Router（Repo A，pure presentation）| SSR/SSG 滿足 NFR-001。| Server-rendered templates—— 較難維護互動式 editor state。|

## 8. 非功能需求如何達成

| NFR | 設計如何滿足 |
|---|---|
| NFR-001（效能：編輯器/網站 <2s，500 同時使用者，p95）| Repo B 單 API monolith 消除跨服務延遲。PostgreSQL on `_status`、`categoryId`、`reviewStatus` 建 index。Repo A 用 SSG + client-side cache。On-demand revalidation 透過 webhook（§6.13）。若 p95 >2s，可選配 Redis 快取。|
| NFR-002（安全：密码杂湊、PII 加密、audit trail）| bcrypt cost 12、HTTPS、JWT httpOnly cookie、PII 加密 at rest。審流與 AI 呼叫皆寫入不可篡改 audit trail（AI_Audit、AI_Generation_Records）。|
| NFR-003（可用性：99.9% uptime）| Repo B 獨立 container + health check，可独立 deploy/scalar。資料庫每日備份。Graceful degradation：AI 不可用時編輯器仍運作。生产前 500 同時使用者 load testing。|
| NFR-004（延展性：10,000 DAU）| PostgreSQL connection pooling（PgBouncer）。Stateless API 允許水平擴展 Repo B。JWT auth 無 session affinity。Repo A 與 Repo B 可各自獨立 scale。|

## 9. Risks & Trade-offs

- **風險：** Payload 的 frontend（Admin + App Router）與 backend 同程序，天然耦合。**緩解：** 以 headless `payload.handler()` 將 backend 獨立為 API（Repo B）；Repo A 用 API Client（§6.3）遠端呼叫。Admin UI 選擇見 §7 / Open Question。
- **風險：** 網站（Repo A）與 API（Repo B）分離，使模板的進程內 draft/live preview 與 on-demand revalidation 無法直接使用。**緩解：** preview 改走 HTTP preview endpoint（Repo B 回 draft 內容，Repo A 渲染）；revalidation 改走 webhook（§6.13）。這是 v3 必須重做的一部分。
- **風險：** 使用 Payload headless 但仍保留其部分 frontend 依賴（如 GraphQL 型別生成）可能洩漏 schema 到前端 repo。**緩解：** 前端只生成「欄位型別」（presentation 所需），不含後端 hooks/access/plugins 邏輯；後端权威設定僅存於 Repo B。
- **風險：** 兩個 repo 的 schema 需要同步（一張表改兩邊都要知道）。**緩解：** 權威 schema 僅在 Repo B；Repo A 透過 API contract（§4）與 generated types 得知資料結構，不複製 backend 設定。
- **風險：** GDPR 合規需要資料匯出/刪除。**緩解：** Users entity 含 soft-delete 與匯出能力；GDPR 是硬限制（PRD §8）。
- **Trade-off：** 採用 Payload headless 而非完全重寫——保留全部後端能力但需處理其 frontend 耦合（headless 已最小化此問題）。若團隊將「豐富的 stock Admin」視為優先於「乾淨的兩 repo 邊界」，則應改用 Express/Fastify + Prisma 完全解耦（見 Open Question）。

## 10. 開放問題

- **回答者：Foan（架構抉擇，僅 Foan/Mike 可定）**
  - **Admin UI 的選擇：** 使用 Payload 的 stock Admin（rich UI，但需 co-located build-time config，會把後端 schema 洩漏進 Repo A、違反兩 repo 邊界）vs. 在 Repo A 自建 lightweight Admin（乾淨邊界，但需自行實作管理介面）？**這是 v3 最關鍵的架構決擇，影響 Repo A 的技術選型與工作量。**
- **回答者：Planner / Foan**
  - 具體的外部 AI 服務提供者為何？（PRD §10：「specific provider TBD」）—— 影響 IAiProvider 實作細節。
- **回答者：Planner / Foan**
  - 除 GDPR 外的區域資料安全要求為何？（PRD §10：GDPR/CCPA）—— 影響加密範圍與資料 residency。
- **回答者：Architect（自行解決）**
  - 前端 SSR framework：採用 Next.js（满足 NFR-001，但需移除模板的 withPayload）。
  - 媒體版本控制：采用独立 media_versions 集合（§5.2），因 PRD「版本控制」指資源多版本歷史。
- **回答者：Architect（自行決定並記明）**
  - Revalidation 機制：因網站與 API 分 repo，改走 webhook（Repo B → Repo A `/api/v1/revalidate`），secret 共用驗證。

---

_Gate 2 核准授權建立 accompanying **Task Breakdown** 中定義的實作任務。本文與 Task Breakdown 一併核准。_
