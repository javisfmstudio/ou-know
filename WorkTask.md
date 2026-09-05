# PJ-0002 — OU Know 任務追蹤

## WorkTask Table

| Work | Title | State | Status | Owner | Reviewer | Priority | Depends On | SubtaskOf | Rejects |
|------|-------|-------|--------|-------|----------|----------|------------|-----------|---------|
| T-001 | Create PRD | Requirement | Done | Planner | Foan | - | - | - | 0 |
| T-002 | Architecture Design | Architecture | Wait for Approval | Architect | Foan | - | T-001 | - | 0 |

## Notes
- Formal PRD at `docs/PRD.md` (approved commit 70c7c34). Gate 1 approved by Foan 2026-09-01 10:13 TST.
- T-002 v1 (commit 62fc77d) superseded per Foan instruction 2026-09-05.
- T-002 v2 (commit 7b0d2d6) superseded per Foan instruction 2026-09-06: split into two independent repos (frontend presentation + backend API), headless Payload, no Next.js DB layer.
- T-002 v3 done by Architect (session timeout, files written to disk) at commit add8a89: `docs/architecture.md` v0.3 (Repo A: Next.js frontend, no DB; Repo B: Payload 3.x headless API, PostgreSQL, Auth/RBAC, AI, Jobs; HTTP API contract between repos) + `docs/task-breakdown.md` v3. Awaiting Gate 2 approval (re-review).
- Open questions pending Foan: (1) external AI provider identity, (2) regional data-security requirements beyond GDPR.
