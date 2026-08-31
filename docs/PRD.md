# PRD: OU Know

## Meta

| Field | Value |
|---|---|
| ProjectId | PJ-0002 |
| Author | Planner |
| Date | 2026-08-31 |
| Version | 0.1 |
| Gate 1 approver | Foan / Mike |
| Status | Draft |

## 1. Summary

OU Know 是一個統一的數位內容管理平台，整合文章發布、資源儲存、使用者安全防護以及 AI 驅動的增強功能。它簡化內容工作流程、集中管理媒體資源、確保安全存取，並提供智慧 SEO 與寫作工具以提升效率與品質。

## 2. Problem & Goal

- **Problem:** 內容管理分散在多個系統中，導致發布效率低落與資源使用不一致。缺乏集中的安全控制機制與手動 SEO 流程。
- **Goal:** 提供一個單一、安全的平台，自動化內容工作流程、集中管理數位資源，並利用 AI 增強內容創作與最佳化。

## 3. Scope

**In scope**
- Content & Article Management: 文章的 CRUD 操作，包含狀態轉換（draft、review、published、archived）、分類與標籤管理。
- Digital Asset Management: 上傳、儲存與管理圖片及其他媒體；與文章關聯；資源版本控制。
- Account & Security: 使用者註冊、登入、密碼重置、基於角色的存取控制（admin、editor、author）。
- Content Intelligence: SEO metadata 生成、AI 輔助內容寫作、語氣調整工具。

**Out of scope (Non-Goals)**
- Mobile application development；僅提供 web-based interface。
- Integration with third-party CRM or marketing automation tools。
- Hardware-specific optimizations or server infrastructure management。
- Custom AI model training；依賴外部服務。

## 4. Users

| User | Needs |
|------|-------|
| Content Author | 使用 SEO 工具建立與編輯文章，管理 draft 狀態，並發布至目標受眾。 |
| Editor | 檢視與核准文章，管理內容分類與標籤，並處理 asset library。 |
| Admin | 管理使用者帳號、權限、系統設定與 audit logs。 |

## 5. User Flows

### UF-001 — Article Creation and Publishing
1. Author 選擇「New Article」→ 顯示表單，包含 title、content、category、tags 與 SEO fields。
2. Author 填寫必填欄位 → 系統驗證輸入（例如 title 必須填寫）。
3. Author 點擊「Save Draft」→ 文章以「draft」狀態儲存。
4. Author 點擊「Publish」→ 系統檢查 status transition rules；若有效，文章狀態變為「published」並對公眾可見。

### UF-002 — Asset Upload and Management
1. User 上傳圖片 → 系統驗證檔案類型（jpg、png、gif、webp）與大小（≤10MB）。
2. 系統將 asset 儲存至 media library；產生 thumbnail preview。
3. User 將 asset 關聯至文章 → 系統更新文章的 media references。

### UF-003 — User Login and Password Management
1. User 輸入 credentials → 系統比對已儲存的 credentials。
2. 登入失敗時，系統顯示錯誤訊息並記錄嘗試次數。
3. User 請求密碼重置 → 系統將 reset link 發送至註冊 email。
4. User 重置密碼 → 系統更新 password hash 並記錄 event。

## 6. Functional Requirements

### FR-001 — Article Creation Form
- **Implements:** UF-001 step 1
- **Requirement:** 系統必須提供表單用於建立新文章，包含 title、content、category、tags 與 SEO metadata 欄位。
- **Acceptance criteria:**
  - Given a user is on the article creation page
    When the form is displayed
    Then all required fields (title, content, category, tags, SEO metadata) are visible and editable
  - Given a user is creating a new article
    When they attempt to save without a title
    Then the system displays an error message
  - Given a user is creating a new article
    When they select a category
    Then the system only allows submission if a valid category is selected

### FR-002 — Article Status Transition
- **Implements:** UF-001 step 3-4
- **Requirement:** Article 狀態必須根據使用者操作在 draft、review 與 published 之間轉換。
- **Acceptance criteria:**
  - Given an article is in "draft" status
    When the author clicks "Save Draft"
    Then the article status remains "draft"
  - Given an article is in "draft" status
    When the author clicks "Publish"
    Then the article status changes to "published"
  - Given an article is in "published" status
    When the author attempts to edit directly
    Then the system requires reverting to "draft" status first

### FR-003 — Asset Upload Validation
- **Implements:** UF-002 step 1-2
- **Requirement:** 系統必須驗證上傳的 asset 是否符合檔案類型與大小要求。
- **Acceptance criteria:**
  - Given a user uploads an image file
    When the file is of type JPG, PNG, GIF, or WebP and ≤10MB
    Then the system accepts the upload and stores it in the media library
  - Given a user uploads a non-image file
    When the system receives the file
    Then the system rejects it with an error message
  - Given a user uploads an image >10MB
    When the system receives the file
    Then the system rejects it with an error message

### FR-004 — Password Hashing
- **Implements:** UF-003 step 4
- **Requirement:** 使用者密碼必須使用 salted hashing algorithm 安全儲存。
- **Acceptance criteria:**
  - Given a user creates a new account or updates their password
    When the password is stored
    Then it is never stored in plaintext
  - Given a user logs in with correct credentials
    When the system verifies the password
    Then it uses a hashed comparison method

### FR-005 — Role-Based Access Control
- **Implements:** UF-003 step 1
- **Requirement:** 系統必須對使用者操作執行基於角色的權限控制。
- **Acceptance criteria:**
  - Given an admin user
    When they access user management features
    Then they can create, edit, and delete accounts
  - Given an author user
    When they attempt to edit another author's article
    Then the system prevents the action
  - Given an editor user
    When they review an article
    Then they can approve or reject it but cannot modify system settings

### FR-006 — SEO Metadata Generation
- **Implements:** Content Intelligence
- **Requirement:** 系統必須根據文章内容生成 SEO metadata（title、description、keywords）。
- **Acceptance criteria:**
  - Given an article is saved
    When the system processes SEO metadata
    Then it auto-populates title and description fields based on content
  - Given a user has manually set SEO metadata
    When they save the article
    Then the system preserves their manual changes

### FR-007 — AI Content Writing Assistance
- **Implements:** Content Intelligence
- **Requirement:** 系統必須根據使用者輸入提供 AI-driven suggestions 用於文章内容。
- **Acceptance criteria:**
  - Given a user is editing an article
    When they request AI suggestions for a section
    Then the system generates relevant content within 10 seconds
  - Given a user selects a tone adjustment option
    When they apply it to a paragraph
    Then the system rewrites the paragraph while maintaining core meaning

### FR-008 — Tone Adjustment Tool
- **Implements:** Content Intelligence
- **Requirement:** 系統必須允許使用者調整寫作內容的語氣（例如 formal、casual）。
- **Acceptance criteria:**
  - Given a user selects a paragraph in the editor
    When they choose "Formal" tone
    Then the system rewrites the paragraph using professional language
  - Given a user selects a paragraph in the editor
    When they choose "Casual" tone
    Then the system rewrites the paragraph using conversational language

## 7. Non-Functional Requirements

| ID | Category | Requirement (measurable) |
|---|---|---|
| NFR-001 | Performance | Page load for article editing must be under 2 seconds at 95th percentile with 500 concurrent users. |
| NFR-002 | Security | All user passwords must be securely hashed; all PII encrypted at rest and in transit. |
| NFR-003 | Availability | System must maintain 99.9% uptime during business hours (9 AM–6 PM, Monday–Friday). |
| NFR-004 | Scalability | System must support up to 10,000 daily active users without performance degradation. |

## 8. Constraints & Dependencies

- **Hard constraints:** 必須符合 GDPR 對於歐盟地區使用者資料處理的規範。
- **Dependencies:** 第三方 AI services 用於 content generation 與 SEO analysis（specific provider TBD）。

## 9. Success Metrics

- 80% 已發布的文章使用 AI-assisted SEO metadata。
- 平均文章建立時間較先前流程減少 30%。
- 95% 的使用者登入在 5 秒內成功。

## 10. Open Questions & Assumptions

- **Assumption:** 所有使用者對 web-based content management systems 有基本熟悉度。
- **Open question:** 用於 content generation 與 SEO analysis 的具體第三方 AI service providers 為何，及其所需的 integration points？
- **Open question:** 不同地區處理使用者資料的具體安全要求為何（例如 GDPR、CCPA）？
