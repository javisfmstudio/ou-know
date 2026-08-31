# PRD: OU Know

## Meta

| 欄位 | 值 |
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

**規劃內（In scope）**
- 內容與文章管理：文章的 CRUD 操作，包含狀態轉換（draft、review、published、archived）、分類與標籤管理。
- 數位資源管理：上傳、儲存與管理圖片及其他媒體；與文章關聯；資源版本控制。
- 帳號與安全：使用者註冊、登入、密碼重置、基於角色的存取控制（admin、editor、author）。
- 內容智慧：SEO metadata 生成、AI 輔助內容寫作、語氣調整工具。

**規劃外（Out of scope / Non-Goals）**
- 行動應用程式開發；僅提供網頁介面。
- 與第三方 CRM 或 marketing automation tools 整合。
- 特定硬體最佳化或伺服器基礎設施管理。
- 客製化 AI model training；依賴外部服務。

## 4. Users

| 使用者 | 需求 |
|------|-------|
| Content Author | 使用 SEO 工具建立與編輯文章，管理 draft 狀態，並發布至目標受眾。 |
| Editor | 檢視與核准文章，管理內容分類與標籤，並處理 asset library。 |
| Admin | 管理使用者帳號、權限、系統設定與 audit logs。 |

## 5. User Flows

### UF-001 — 文章建立與發布
1. Author 選擇「New Article」→ 顯示表單，包含 title、content、category、tags 與 SEO fields。
2. Author 填寫必填欄位 → 系統驗證輸入（例如 title 必須填寫）。
3. Author 點擊「Save Draft」→ 文章以「draft」狀態儲存。
4. Author 點擊「Publish」→ 系統檢查 status transition rules；若有效，文章狀態變為「published」並對公眾可見。

### UF-002 — 資源上傳與管理
1. User 上傳圖片 → 系統驗證檔案類型（jpg、png、gif、webp）與大小（≤10MB）。
2. 系統將 asset 儲存至 media library；產生 thumbnail preview。
3. User 將 asset 關聯至文章 → 系統更新文章的 media references。

### UF-003 — 使用者登入與密碼管理
1. User 輸入 credentials → 系統比對已儲存的 credentials。
2. 登入失敗時，系統顯示錯誤訊息並記錄嘗試次數。
3. User 請求密碼重置 → 系統將 reset link 發送至註冊 email。
4. User 重置密碼 → 系統更新 password hash 並記錄 event。

## 6. Functional Requirements

### FR-001 — 文章建立表單
- **Implements:** UF-001 step 1
- **Requirement:** 系統必須提供表單用於建立新文章，包含 title、content、category、tags 與 SEO metadata 欄位。
- **Acceptance criteria:**
  - Given 使用者在文章建立頁面
    When 表單顯示時
    Then 所有必填欄位（title、content、category、tags、SEO metadata）皆可見且可編輯
  - Given 使用者建立新文章時
    When 嘗試儲存但未填寫 title 時
    Then 系統顯示錯誤訊息
  - Given 使用者建立新文章時
    When 選擇 category 時
    Then 系統僅在接受有效 category 時允許送出

### FR-002 — 文章狀態轉換
- **Implements:** UF-001 step 3-4
- **Requirement:** Article 狀態必須根據使用者操作在 draft、review 與 published 之間轉換。
- **Acceptance criteria:**
  - Given 文章處於「draft」狀態時
    When 作者點擊「Save Draft」時
    Then 文章狀態維持「draft」
  - Given 文章處於「draft」狀態時
    When 作者點擊「Publish」時
    Then 文章狀態變更為「published」
  - Given 文章處於「published」狀態時
    When 作者嘗試直接編輯時
    Then 系統要求先還原為「draft」狀態

### FR-003 — 資源上傳驗證
- **Implements:** UF-002 step 1-2
- **Requirement:** 系統必須驗證上傳的 asset 是否符合檔案類型與大小要求。
- **Acceptance criteria:**
  - Given 使用者上傳圖片檔案時
    When 檔案類型為 JPG、PNG、GIF 或 WebP 且大小 ≤10MB 時
    Then 系統接受上傳並儲存至 media library
  - Given 使用者上傳非圖片檔案時
    When 系統收到檔案時
    Then 系統以錯誤訊息拒絕
  - Given 使用者上傳超過 10MB 的圖片時
    When 系統收到檔案時
    Then 系統以錯誤訊息拒絕

### FR-004 — 密碼雜湊
- **Implements:** UF-003 step 4
- **Requirement:** 使用者密碼必須使用 salted hashing algorithm 安全儲存。
- **Acceptance criteria:**
  - Given 使用者建立新帳號或更新密碼時
    When 密碼儲存時
    Then 絕不以 plaintext 形式儲存
  - Given 使用者以正確的 credentials 登入時
    When 系統驗證密碼時
    Then 系統使用 hashed comparison method 進行比對

### FR-005 — 基於角色的存取控制
- **Implements:** UF-003 step 1
- **Requirement:** 系統必須對使用者操作執行基於角色的權限控制。
- **Acceptance criteria:**
  - Given 管理者使用者時
    When 存取使用者管理功能時
    Then 可以建立、編輯與刪除帳號
  - Given 作者使用者時
    When 嘗試編輯其他作者的文章時
    Then 系統阻止此操作
  - Given 編輯者使用者時
    When 檢視文章時
    Then 可以核准或拒絕，但無法修改系統設定

### FR-006 — SEO Metadata 生成
- **Implements：** 內容智慧（Content Intelligence）
- **Requirement:** 系統必須根據文章內容生成 SEO metadata（title、description、keywords）。
- **Acceptance criteria:**
  - Given 文章儲存時
    When 系統處理 SEO metadata 時
    Then 系統會根據內容自動填入 title 與 description 欄位
  - Given 使用者已手動設定 SEO metadata 時
    When 儲存文章時
    Then 系統保留其手動設定

### FR-007 — AI 內容寫作輔助
- **Implements：** 內容智慧（Content Intelligence）
- **Requirement:** 系統必須根據使用者輸入提供 AI-driven suggestions 用於文章內容。
- **Acceptance criteria:**
  - Given 使用者編輯文章時
    When 要求 AI 針對某個段落提供建議時
    Then 系統於 10 秒內產生相關內容
  - Given 使用者選擇語氣調整選項時
    When 套用至某個段落時
    Then 系統重寫該段落，同時維持核心語意

### FR-008 — 語氣調整工具
- **Implements：** 內容智慧（Content Intelligence）
- **Requirement:** 系統必須允許使用者調整寫作內容的語氣（例如 formal、casual）。
- **Acceptance criteria:**
  - Given 使用者在編輯器中選取段落時
    When 選擇「Formal」語氣時
    Then 系統以專業用語重寫該段落
  - Given 使用者在編輯器中選取段落時
    When 選擇「Casual」語氣時
    Then 系統以口語化用語重寫該段落

## 7. Non-Functional Requirements

| ID | 類別 | 需求（可量測） |
|---|---|---|
| NFR-001 | 效能 | 文章編輯頁面載入時間在 500 個同時使用者下，95th percentile 必須低於 2 秒。 |
| NFR-002 | 安全 | 所有使用者密碼必須安全雜湊；所有 PII 在儲存與傳輸時皆需加密。 |
| NFR-003 | 可用性 | 系統在營運時間內（週一至週五上午 9 點至下午 6 點）必須維持 99.9% uptime。 |
| NFR-004 | 延展性 | 系統必須支援最多 10,000 名每日活躍使用者，且不影響效能。 |

## 8. Constraints & Dependencies

- **Hard constraints:** 必須符合 GDPR 對於歐盟地區使用者資料處理的規範。
- **Dependencies：** 第三方 AI 服務用於內容產生（content generation）與 SEO 分析（SEO analysis）（specific provider TBD）。

## 9. Success Metrics

- 80% 已發布的文章使用 AI 輔助 SEO metadata。
- 平均文章建立時間較先前流程減少 30%。
- 95% 的使用者登入在 5 秒內成功。

## 10. Open Questions & Assumptions

- **Assumption：** 所有使用者對網頁內容管理系統有基本熟悉度。
- **Open question：** 用於內容產生（content generation）與 SEO 分析（SEO analysis）的具體第三方 AI 服務提供者為何，及其所需的 integration points？
- **Open question：** 不同地區處理使用者資料的具體安全要求為何（例如 GDPR、CCPA）？
