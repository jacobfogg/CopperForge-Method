# Expected source-file inventory

The 15 files [methodology-bootstrap](../SKILL.md) workflow step 5 verifies are
present in the `methodology/` tree at the resolved commit SHA before any disk
write. Drawn from `jacobfogg/CopperForge-Method` `default_branch=main` as of
2026-04-26.

## Closed file list (15 paths)

### Company-root files (3)

| Source path | Destination |
|---|---|
| `methodology/COMPANY.md` | `companies/{id}/COMPANY.md` (single copy) |
| `methodology/PLATFORM.md` | `companies/{id}/PLATFORM.md` (single copy) |
| `methodology/CODING_STANDARDS.md` | `companies/{id}/CODING_STANDARDS.md` (single copy) |

### Per-role files (12, fanned out per matching agent count)

| Source path | Destination per matching agent |
|---|---|
| `methodology/roles/CEO/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/CEO/HEARTBEAT.md` | `companies/{id}/agents/{agentId}/instructions/HEARTBEAT.md` |
| `methodology/roles/CEO/SOUL.md` | `companies/{id}/agents/{agentId}/instructions/SOUL.md` |
| `methodology/roles/CEO/TOOLS.md` | `companies/{id}/agents/{agentId}/instructions/TOOLS.md` |
| `methodology/roles/CTO/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/CTO/HEARTBEAT.md` | `companies/{id}/agents/{agentId}/instructions/HEARTBEAT.md` |
| `methodology/roles/CTO/SOUL.md` | `companies/{id}/agents/{agentId}/instructions/SOUL.md` |
| `methodology/roles/BA/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/Auditor/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/QAManager/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/QA/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |
| `methodology/roles/Dev/AGENTS.md` | `companies/{id}/agents/{agentId}/instructions/AGENTS.md` |

## Out of scope (explicitly NOT required)

- `methodology/README.md` — the methodology repo's own README is non-load-bearing
  framing per BA research (COPAAA-59) and is **explicitly optional** per
  [COPAAA-58 plan §9.6](/COPAAA/issues/COPAAA-58#document-plan) (CTO recommendation:
  skip in v1.0; deploy only if a future use case emerges). The install operation
  MUST NOT fetch or write `README.md` and MUST NOT include it in the 15-file
  expected set.

## Maintenance discipline

If the methodology repo gains or loses a `methodology/roles/<Role>/<file>.md`
file in a future release tag, this inventory plus
[`role-normalization.md`](role-normalization.md) are the two reference files
to update before bumping the install operation's expected count. Source-file
adds also need a coordinated entry in the role-normalization table when a
brand-new role dir is introduced.
