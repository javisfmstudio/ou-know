# PJ-0002 — OU Know 任務追蹤

## WorkTask Table

| Work | Title | State | Status | Owner | Reviewer | Priority | Depends On | SubtaskOf | Rejects |
|------|-------|-------|--------|-------|----------|----------|------------|-----------|---------|
| T-001 | Create PRD | Requirement | Done | Planner | Foan | - | - | - | 0 |
| T-002 | Architecture Design | Architecture | Wait for Approval | Architect | Foan | - | T-001 | - | 0 |

## Notes
- Formal PRD at `docs/PRD.md` (approved commit 70c7c34). Gate 1 approved by Foan 2026-09-01 10:13 TST.
- T-002 v1 (commit 62fc77d) superseded per Foan instruction 2026-09-05: re-design to maximize features of `ouknow_website` (Payload 3.x template) + original PJ-0002 PRD, keep proprietary API, extended DB schema.
- T-002 v2 done by Architect at commit 7b0d2d6: `docs/architecture.md` v0.2 (Payload 3.x monolith + proprietary API layer, merged DB schema incl. AI/audit tables) + `docs/task-breakdown.md` v2. Awaiting Gate 2 approval (re-review).
- Open questions pending Foan: (1) external AI provider identity, (2) regional data-security requirements beyond GDPR.
