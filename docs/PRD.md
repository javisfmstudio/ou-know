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

OU Know is a unified platform for digital content management, integrating article publishing, asset storage, user security, and AI-driven enhancements. It streamlines content workflows, centralizes media assets, ensures secure access, and provides intelligent SEO and writing tools to improve efficiency and quality.

## 2. Problem & Goal

- **Problem:** Content management is fragmented across multiple systems, leading to inefficiencies in publishing and inconsistent asset usage. There's a lack of centralized security controls and manual SEO processes.
- **Goal:** Provide a single, secure platform that automates content workflows, centralizes digital assets, and leverages AI to enhance content creation and optimization.

## 3. Scope

**In scope**
- Content & Article Management: CRUD operations for articles, including status transitions (draft, review, published, archived), category and tag management.
- Digital Asset Management: Upload, store, and manage images and other media; associate with articles; versioning of assets.
- Account & Security: User registration, login, password reset, role-based access control (admin, editor, author).
- Content Intelligence: SEO metadata generation, AI-assisted content writing, tone adjustment tools.

**Out of scope (Non-Goals)**
- Mobile application development; only web-based interface.
- Integration with third-party CRM or marketing automation tools.
- Hardware-specific optimizations or server infrastructure management.
- Custom AI model training; relies on external services.

## 4. Users

| User | Needs |
|------|-------|
| Content Author | Create and edit articles with SEO tools, manage draft status, and publish to target audiences. |
| Editor | Review and approve articles, manage content categories and tags, and handle asset library. |
| Admin | Manage user accounts, permissions, system settings, and audit logs. |

## 5. User Flows

### UF-001 — Article Creation and Publishing
1. Author selects "New Article" → form appears with title, content, category, tags, and SEO fields.
2. Author fills in required fields → system validates inputs (e.g., title must be present).
3. Author clicks "Save Draft" → article saved with status "draft".
4. Author clicks "Publish" → system checks status transition rules; if valid, article status becomes "published" and visible to public.

### UF-002 — Asset Upload and Management
1. User uploads image → system validates file type (jpg, png, gif, webp) and size (≤10MB).
2. System stores asset in media library; generates thumbnail preview.
3. User associates asset with article → system updates article's media references.

### UF-003 — User Login and Password Management
1. User enters credentials → system checks against stored credentials.
2. On failed login, system displays error and tracks attempts.
3. User requests password reset → system sends reset link to registered email.
4. User resets password → system updates password hash and logs event.

## 6. Functional Requirements

### FR-001 — Article Creation Form
- **Implements:** UF-001 step 1
- **Requirement:** System must provide a form for creating new articles with fields for title, content, category, tags, and SEO metadata.
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
- **Requirement:** Article status must transition between draft, review, and published based on user actions.
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
- **Requirement:** System must validate uploaded assets for file type and size.
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
- **Requirement:** User passwords must be securely stored using a salted hashing algorithm.
- **Acceptance criteria:**
  - Given a user creates a new account or updates their password
    When the password is stored
    Then it is never stored in plaintext
  - Given a user logs in with correct credentials
    When the system verifies the password
    Then it uses a hashed comparison method

### FR-005 — Role-Based Access Control
- **Implements:** UF-003 step 1
- **Requirement:** System must enforce role-based permissions for user actions.
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
- **Requirement:** System must generate SEO metadata (title, description, keywords) based on article content.
- **Acceptance criteria:**
  - Given an article is saved
    When the system processes SEO metadata
    Then it auto-populates title and description fields based on content
  - Given a user has manually set SEO metadata
    When they save the article
    Then the system preserves their manual changes

### FR-007 — AI Content Writing Assistance
- **Implements:** Content Intelligence
- **Requirement:** System must provide AI-driven suggestions for article content based on user input.
- **Acceptance criteria:**
  - Given a user is editing an article
    When they request AI suggestions for a section
    Then the system generates relevant content within 10 seconds
  - Given a user selects a tone adjustment option
    When they apply it to a paragraph
    Then the system rewrites the paragraph while maintaining core meaning

### FR-008 — Tone Adjustment Tool
- **Implements:** Content Intelligence
- **Requirement:** System must allow users to adjust the tone of written content (e.g., formal, casual).
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

- **Hard constraints:** Must comply with GDPR for user data processing in EU regions.
- **Dependencies:** Third-party AI services for content generation and SEO analysis (specific provider TBD).

## 9. Success Metrics

- 80% of published articles use AI-assisted SEO metadata.
- Average article creation time reduced by 30% compared to previous process.
- 95% of user logins succeed within 5 seconds.

## 10. Open Questions & Assumptions

- **Assumption:** All users have basic familiarity with web-based content management systems.
- **Open question:** What are the specific third-party AI service providers to be used for content generation and SEO analysis, and their required integration points?
- **Open question:** What are the exact security requirements for handling user data in different regions (e.g., GDPR, CCPA)?