# 架構設計：OU Know

> **撰寫規範（Architect 必須遵守）：**
> 1. 本文回答 PRD 未開的 **how**。不重新開啟或重新協商需求 —— 若需求有誤或不清楚，提出澄清，不要靜默重新設計。
> 2. 每個設計決策都追溯至需求（FR-XXX / NFR-XXX）。若 component 未對應到任何需求，以 enabling 工作說明或刪除此 component。
> 3. 明確定義 component 之間的 contract —— 特別是 Dev-be / Dev-fe 邊界。兩個 agent 對著模糊介面開發，會做出不相容的兩半。
> 4. 說明 trade-off 與 rejected alternatives。沒有 trade-off 的設計是沒有思考過的設計。
> 5. 不在本文分配或排程工作 —— 那是 Task Breakdown 的事。本文是「它長什麼樣子」，不是「誰做什麼、何時做」。
> _(提交前刪除此區塊。)_

## Meta

| 欄位 | 值 |
|---|---|
| ProjectId | PJ-0002 |
| 作者 | Architect |
| 日期 | 2026-09-01 |
| 版本 | 0.1 |
| 基於 PRD 版本 | 0.1 |
| Gate 2 核准者 | Foan / Mike |
| 狀態 | Draft |

---

## 1. Approach

OU Know 是一個統一的數位內容管理平台，包含四個緊密耦合的領域：文章管理、數位資源、使用者帳號、以及 AI 驅動的內容智慧。解決方案為**單一模擬式後端（monolithic backend）**搭配**現代化 SPA 前端**，共用統一的資料庫與 session 層。最重要的設計決策是**不將 AI 內容智慧層拆為獨立服務** —— 它與文章編輯緊密耦合，在此專案規模下獨立部署是不必要的 overhead。

## 2. 架構風格與規範

**風格：Monolith（已論證）**

| 面向 | 決策 | 為何適合此專案規模 |
|---|---|---|
| 架構風格 | Monolith | 10,000 DAU、500 同時使用者、單一支團隊、四個緊密耦合領域。無獨立部署需求。Monolith 消除了跨服務延遲、分散式交易複雜度與營運 overhead。 |

**所有程式碼必須遵守的規範：**
- **Clean code 基線（不可妥協）：** 清晰的命名、單一職責函式、無死碼或重複程式碼、可讀的結構。
- **程式碼組織：** 分層架構 —— `controllers`（HTTP request/response）→ `services`（business logic）→ `repositories`（data access）→ `models`（entity definitions）。禁止 controller 跳過 service 直接呼叫 repository。
- **命名規範：** model 用 PascalCase，API 欄位用 camelCase，檔案名用 kebab-case。路由路徑：`/api/v1/<resource>`。
- **錯誤處理：** 所有錯誤遵循結構化 `ApiError` 格式 `{code, message, details}`。不得將 raw exception 洩漏至 HTTP response。

## 3. Component Map

| Component | 職責 | 服務於 | 詳細 |
|---|---|---|---|
| Auth | 使用者註冊、登入、密碼重設、RBAC 執行 | FR-003, FR-004, FR-005, NFR-002 | §6.1 |
| Article | 文章 CRUD、狀態轉換、分類、標籤 | FR-001, FR-002 | §6.2 |
| Media | 上傳驗證、儲存、縮圖產生、文章關聯 | FR-003, UF-002 | §6.3 |
| Content Intelligence | SEO metadata 產生、AI 輔助寫作、語氣調整 | FR-006, FR-007, FR-008 | §6.4 |
| Frontend SPA | 文章編輯器、媒體庫、使用者管理 UI | 所有 FR（presentation layer） | §6.5 |

## 4. System-Wide Interfaces

| From → To | Contract（endpoint / event / data shape） | Notes |
|---|---|---|
| Frontend → Backend | `POST /api/v1/auth/register` `{email, password, role?}` → `{user, token}` | Password 必須 ≥8 字元 |
| Frontend → Backend | `POST /api/v1/auth/login` `{email, password}` → `{user, token}` | 失敗回 401，5 次失敗後 lockout |
| Frontend → Backend | `POST /api/v1/auth/password-reset/request` `{email}` → `{204}` | 透過 email 發送 reset link |
| Frontend → Backend | `POST /api/v1/auth/password-reset/confirm` `{token, newPassword}` → `{204}` | Token 有效 1 小時，單次使用 |
| Frontend → Backend | `GET/POST/PUT/DELETE /api/v1/articles[/:id]` | 文章 CRUD 搭配 role-based access |
| Frontend → Backend | `POST /api/v1/articles/:id/publish` → `{article}` | 狀態轉換：draft → published |
| Frontend → Backend | `POST /api/v1/articles/:id/draft` → `{article}` | 狀態轉換：published → draft（revert） |
| Frontend → Backend | `POST /api/v1/media/upload` (multipart) → `{media, thumbnail}` | 最大 10MB，類型驗證 |
| Frontend → Backend | `GET /api/v1/media[/:id]` → `{media}` | 列出 / 取得媒體項目 |
| Frontend → Backend | `POST /api/v1/articles/:id/media` `{mediaIds[]}` → `{article}` | 將媒體關聯至文章 |
| Frontend → Backend | `POST /api/v1/content/seo/generate` `{articleId}` → `{title, description, keywords}` | 自動產生 SEO metadata |
| Frontend → Backend | `POST /api/v1/content/ai/suggest` `{articleId, paragraph?, focus?}` → `{suggestions[]}` | AI 寫作輔助 |
| Frontend → Backend | `POST /api/v1/content/tone/adjust` `{articleId, paragraph, tone}` → `{rewritten}` | 語氣調整：formal/casual |

## 5. Data Model

### Users
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| email | VARCHAR(255) | Unique, indexed |
| password_hash | VARCHAR(255) | bcrypt/argon2 salted hash |
| role | ENUM('admin','editor','author') | RBAC role |
| reset_token | VARCHAR(255) | NULL 表示無待處理的 reset |
| reset_token_expires | TIMESTAMP | NULL 表示無待處理的 reset |
| failed_attempts | INT | 成功登入後重設為 0 |
| locked_until | TIMESTAMP | NULL 表示未 lock |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Articles
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| title | VARCHAR(500) | 必填 |
| content | TEXT | Markdown |
| status | ENUM('draft','review','published','archived') | |
| author_id | UUID (FK → Users) | |
| category_id | UUID (FK → Categories) | |
| seo_title | VARCHAR(255) | NULL = 自動產生 |
| seo_description | TEXT | NULL = 自動產生 |
| seo_keywords | JSON | NULL = 自動產生 |
| published_at | TIMESTAMP | NULL 表示未發表 |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### Media
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| original_filename | VARCHAR(255) | |
| stored_filename | VARCHAR(255) | Unique, 系統產生 |
| mime_type | VARCHAR(100) | 上傳時驗證 |
| size_bytes | BIGINT | |
| thumbnail_url | VARCHAR(500) | 上傳時產生 |
| uploaded_by | UUID (FK → Users) | |
| created_at | TIMESTAMP | |

### Article_Media（junction）
| 欄位 | 型別 | 備註 |
|---|---|---|
| article_id | UUID (FK → Articles) | |
| media_id | UUID (FK → Media) | |

### Categories
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(100) | Unique |
| slug | VARCHAR(100) | Unique, URL-friendly |

### Tags
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(100) | Unique |

### Article_Tags（junction）
| 欄位 | 型別 | 備註 |
|---|---|---|
| article_id | UUID (FK → Articles) | |
| tag_id | UUID (FK → Tags) | |

### Password_Reset_Tokens（audit log, referenced by Users.reset_token）
| 欄位 | 型別 | 備註 |
|---|---|---|
| id | UUID | Primary key |
| user_id | UUID (FK → Users) | |
| token_hash | VARCHAR(255) | 儲存時 hash |
| created_at | TIMESTAMP | 1 小時後過期 |

## 6. Component Detail

### 6.1 Auth（認證與存取控制）
- **Responsibility:** 使用者註冊、認證、密碼管理、RBAC 執行。
- **Serves:** FR-003, FR-004, FR-005, NFR-002
- **Interfaces:**
  - Inbound: `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/password-reset/request`, `POST /api/v1/auth/password-reset/confirm`
  - Outbound: Email service（用於發送 reset link）
- **Data it owns / touches:** Users table, Password_Reset_Tokens table
- **Applicable NFRs:**
  - NFR-002：bcrypt（cost factor 12）用於密碼雜湊；所有 auth traffic 使用 TLS；password_hash 以 salted hash 儲存，永不以 plaintext 形式儲存。
  - NFR-001：登入回應 <100ms p95（indexed email lookup，無重度計算）。
- **Depends on:** Email service（外部）, Frontend SPA（login/register pages）
- **Key decisions:**
  - JWT access tokens（15 min TTL）+ refresh tokens（7 day TTL，以 httpOnly cookies 儲存）。
  - Account lockout：5 次連續失敗 → lock 15 分鐘（FR-003 step 2：「記錄嘗試次數」）。
  - RBAC middleware：每個 protected route 檢查 `req.user.role` 是否符合路由級別的 role 需求。

### 6.2 Article（文章管理）
- **Responsibility:** 文章 CRUD、狀態轉換、分類、標籤。
- **Serves:** FR-001, FR-002
- **Interfaces:**
  - Inbound: `GET/POST/PUT/DELETE /api/v1/articles[/:id]`, `POST /api/v1/articles/:id/publish`, `POST /api/v1/articles/:id/draft`, `POST /api/v1/articles/:id/media`
  - Outbound: Content Intelligence service（publish 時自動產生 SEO）, Media service（媒體關聯參考）
- **Data it owns / touches:** Articles, Categories, Tags, Article_Media, Article_Tags
- **Applicable NFRs:**
  - NFR-001：文章編輯器頁面載入 <2s（500 同時使用者）—— 透過文章資料的 server-side rendering 搭配分頁達成，資料庫 on `status`, `author_id`, `category_id` 建 index。
  - NFR-004：Schema 支援 10,000 DAU —— articles table 建 index 以利高效過濾。
- **Depends on:** Auth（RBAC）, Content Intelligence（SEO auto-fill）, Media（媒體關聯）
- **Key decisions:**
  - 狀態轉換表在 service layer 強制執行（不只靠 DB constraint）：
    - `draft` → `review` / `published` / `draft`（no-op）
    - `review` → `draft` / `published`
    - `published` → `draft`（僅 revert；編輯需先 revert —— FR-002 驗收標準）
    - `archived` → `draft`（僅 admin）
  - `published_at` 在轉換為 `published` 時設定，revert 時清除。
  - Category/Tag 列表作為文章表單 payload 的一部分回傳（一次 round trip 渲染表單 —— 支援 NFR-001）。

### 6.3 Media（數位資源管理）
- **Responsibility:** 上傳驗證、儲存、縮圖產生、媒體擷取。
- **Serves:** FR-003, UF-002
- **Interfaces:**
  - Inbound: `POST /api/v1/media/upload` (multipart), `GET /api/v1/media[/:id]`
  - Outbound: File system / cloud storage（文章上傳）, Image processing library（縮圖產生）
- **Data it owns / touches:** Media, Article_Media
- **Applicable NFRs:**
  - NFR-003：媒體儲存冗餘（備份）支援 uptime SLA。
  - NFR-002：儲存檔案路徑而非檔案內容於 DB；存取透過 Auth middleware 控制。
- **Depends on:** Auth（authenticated upload）, Article（媒體關聯 endpoint）
- **Key decisions:**
  - 上傳驗證：MIME type 檢查（magic bytes）+ extension 檢查 + size ≤10MB。無效時以 400 拒絕（FR-003）。
  - 縮圖：上傳時同步產生，使用 sharp/resize 至最大 320px 尺寸。
  - 儲存：local filesystem 位於 `uploads/`，以 hashed directory 結構（`uploads/ab/cd/...`）避免單一目錄 inode 限制。（若需擴展至單一伺服器以外則 TBD cloud storage。）
  - Filename：UUID + 原始 extension，避免碰撞及暴露原始檔名。

### 6.4 Content Intelligence（內容智慧）
- **Responsibility:** SEO metadata 產生、AI 輔助寫作、語氣調整。
- **Serves:** FR-006, FR-007, FR-008
- **Interfaces:**
  - Inbound: `POST /api/v1/content/seo/generate`, `POST /api/v1/content/ai/suggest`, `POST /api/v1/content/tone/adjust`
  - Outbound: External AI service（provider TBD —— FR-007/008 取決於此）
- **Data it owns / touches:** Articles（讀取內容、寫入 SEO 欄位），無新 table
- **Applicable NFRs:**
  - NFR-001：AI 建議回應 <10s（FR-007 驗收標準）。外部 AI 呼叫外層包覆 timeout。
  - NFR-004：外部 AI 呼叫為 stateless；每使用者 rate limiting 防止濫用。
- **Depends on:** Auth（使用者必須登入）, Article（文章內容擷取）, External AI service
- **Key decisions:**
  - **不要拆為 microservice。** Content Intelligence 在文章編輯期間被呼叫；network hop 到獨立服務會增加延遲到已對延遲敏感的用戶流程。
  - 外部 AI 呼叫外層包覆 timeout（建議 10s、語氣調整 5s），搭配 graceful degradation —— 若 AI 不可用，回傳 `408 Request Timeout` 並提示重試。
  - SEO metadata：若使用者已手動設定任何 SEO 欄位（title/description/keywords），該欄位的自動產生即跳過（FR-006：「保留其手動設定」）。
  - AI 整合點：單一 adapter 介面（`IAiProvider`）搭配具體 provider 實現。可替換而不需更動 business logic。

### 6.5 Frontend SPA（網頁介面）
- **Responsibility:** 文章編輯器、媒體庫、使用者管理 UI、認證流程。
- **Serves:** 所有 FR（presentation layer）
- **Interfaces:**
  - Outbound: 所有 `api/v1/*` endpoints（後端通訊的唯一 truth source）
  - Inbound: 瀏覽器 HTTP 請求（static asset 由 backend 或 CDN 提供）
- **Data it owns / touches:** 無（stateless，透過 API 通訊）
- **Applicable NFRs:**
  - NFR-001：文章編輯器頁面 <2s 載入 —— 透過頁面載入時 initial data prefetch（articles、categories、tags、media list）達成，不使用 lazy-loading。
  - NFR-003：SPA 從 static assets 提供；backend 僅處理動態 API 呼叫。
- **Depends on:** Backend API（所有資料存取）
- **Key decisions:**
  - SPA framework：React（或等效）搭配 server-side rendering 以滿足 NFR-001 初始載入時間。
  - State management：client-side cache 搭配 editor 的 optimistic updates；狀態變更時 refetch。
  - Authentication：JWT 以 httpOnly cookie 儲存（非 localStorage）以防 XSS token 竊取。
  - Editor：rich-text Markdown editor 搭配內嵌 AI suggestion panel（FR-007, FR-008）。

## 7. Technology Decisions

| 決策 | 選擇 | 理由 | 拒絕的替代方案 |
|---|---|---|---|
| Backend framework | Node.js + Express（或 NestJS） | 符合團隊現有 stack 假設；快速迭代；影像處理（sharp）、JWT、驗證的生態系完整。 | Python/Django —— 此範圍較沉重；Ruby on Rails —— AI 整合生態系較小。 |
| Database | PostgreSQL | ACID compliance 用於 RBAC 和文章狀態轉換；JSONB 用於 SEO keywords；成熟生態系。 | SQLite —— 500 同時使用者不足（NFR-001）。MongoDB —— 無 ACID 用於交易型 auth 操作。 |
| Session / Auth | JWT（access）+ refresh token（httpOnly cookie） | Stateless access tokens 用於 API；cookie 中的 refresh tokens 防止 XSS token 竊取。 | Server-side sessions —— 此規模下不必要的 state；localStorage tokens —— XSS vulnerable。 |
| Password hashing | bcrypt（cost factor 12） | FR-004 要求 salted hashing；bcrypt 經得起考驗且符合 NFR-002。 | Argon2id —— 安全性更好但 500 同時登入時 CPU 成本較高；PBKDF2 —— 對 GPU 攻擊較弱。 |
| File storage | Local filesystem（uploads/ with hashed directories） | 簡單、快速，適合 10K DAU 單一伺服器部署。 | S3/cloud storage —— 目前規模過頭；增加營運複雜度。 |
| Thumbnail generation | sharp（Node.js 影像處理） | 快速、compiled、不依賴系統 library。 | GraphicsMagick/ImageMagick —— 較重的系統依賴。 |
| AI integration | Adapter pattern with pluggable provider | FR-007/008 指定「TBD provider」；adapter 允許無程式碼變更下替換。 | Hard-coded provider call —— 在選擇 provider 前就鎖定。 |
| Frontend | React SPA with SSR | 符合 NFR-001 初始載入時間；rich-text editor 和 AI UI 模式的生態系龐大。 | Server-rendered HTML templates —— 較難維護互動式 editor state。 |

## 8. 非功能需求如何達成

| NFR | 設計如何滿足 |
|---|---|
| NFR-001（效能：編輯器頁面 <2s，500 同時使用者，p95） | 單一 monolith 消除跨服務延遲。文章編輯器資料以單一 API 呼叫預先擷取。PostgreSQL on `status`, `author_id`, `category_id` 建 index。SSR 用於初始頁面載入。若基準測試顯示 p95 >2s，可選配 Redis 快取經常存取的文章資料。 |
| NFR-002（安全：密碼雜湊、PII 加密） | 所有密碼使用 bcrypt cost 12。所有流量使用 TLS。密碼雜湊僅以 salted values 儲存。JWT refresh tokens 在 httpOnly cookies。PII（email、reset tokens）以 PostgreSQL pgcrypto 或 application-level encryption 加密 at rest。GDPR 合規：資料匯出/刪除 endpoints、同意紀錄。 |
| NFR-003（可用性：營運時間內 99.9% uptime） | Monolith 部署於單一 container 搭配 health check。上傳檔案備份。資料庫每日備份。Graceful degradation：若 AI service 當機，編輯器仍正常運作（SEO/手動內容）。生產前在 500 同時使用者下進行 load testing。 |
| NFR-004（延展性：10,000 DAU） | PostgreSQL connection pooling（必要时 PgBouncer）。靜態媒體可透過 CDN 提供（選用）。Stateless API 設計允許水平擴展 backend（如需）。無需 session affinity（JWT-based auth）。 |

## 9. Risks & Trade-offs

- **風險：** 外部 AI 服務提供者尚未選擇（PRD §10 開放問題）—— 阻擋 Content Intelligence 實作。**緩解：** Adapter 介面設計為 provider-agnostic；開發期間可使用 mock 實現。
- **風險：** Local file storage 無法擴展至單一伺服器以外。**緩解：** Storage adapter pattern（類似 AI adapter）；需要時無業務邏輯變更即可換至 S3-compatible storage。
- **風險：** 若範圍成長，Monolith 可能難以維護。**緩解：** Clean layered architecture（controllers → services → repositories）強制邊界，使未來分解更容易。
- **風險：** GDPR 合規需要資料匯出/刪除 —— PRD 的 FR 中未明確列出。**緩解：** 設計 Users entity 含 soft-delete 和匯出能力；GDPR 是硬限制（PRD §8），即使未列為 FR 也必須實作。

## 10. 開放問題

- **回答者：** Planner / Foan
  - 具體的外部 AI 服務提供者為何？（PRD §10：「specific provider TBD」）—— 影響 adapter 實作細節。
- **回答者：** Planner / Foan
  - 除 GDPR 外的區域資料安全要求為何？（PRD §10：GDPR/CCPA）—— 影響加密範圍和資料 residency 決策。
- **回答者：** Architect（自行解決）
  - Frontend SSR framework 選擇（Next.js vs. custom SSR）—— 將於實作期間根據團隊熟悉度決定。

---

_Gate 2 核准授權建立 accompanying **Task Breakdown** 中定義的實作任務。本文與 Task Breakdown 一併核准。_
