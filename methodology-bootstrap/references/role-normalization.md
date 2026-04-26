# Role-enum normalization table

Canonical mapping of `methodology/roles/<Role>/` source directories in
[jacobfogg/CopperForge-Method](https://github.com/jacobfogg/CopperForge-Method) to
the Paperclip agent `role` enum (and the `urlKey` tiebreaker where the enum is
ambiguous). Used by every operation in [`methodology-bootstrap`](../SKILL.md)
to fan a single source file out to all matching agents in a target company.

This table is the **single point of update** when Paperclip-core changes the
role enum or when this CopperForge company's role-record values are migrated.
Duplicating the table inline in the skill body or in any other file violates
the SKILL.md quality bar.

The table below is reproduced verbatim from the [COPAAA-58 plan §0.1](/COPAAA/issues/COPAAA-58#document-plan)
(rev 2, CEO-ratified 2026-04-26) so that a maintainer reading this reference
file alone has the full normalization contract.

## Table

| Methodology source dir | Match rule (primary → tiebreaker) | This CopperForge company's matched agents (verified 2026-04-26) |
|---|---|---|
| `roles/CEO/` | `role=ceo` | CEO (`52dde9ed-…`) |
| `roles/CTO/` | `role=cto` | CTO (`74308178-…`) |
| `roles/BA/` | `role=researcher` | BA (`dc990f28-…`) |
| `roles/Auditor/` | `role=audit` (preferred per CEO comment); fall back to `role=general` if `audit` enum absent (live data shows `general` here — install-time hard-fail catches mismatches) | Auditor (`19c8fff8-…`, currently `role=general`) |
| `roles/QAManager/` | `role=qa_manager` (preferred per CEO comment); fall back to `role=qa` AND `urlKey LIKE 'qam%'` if `qa_manager` enum absent | QAM (`08960d7b-…`, currently `role=qa, urlKey=qam`) |
| `roles/QA/` | `role=qa` AND `urlKey NOT LIKE 'qam%'` (i.e., excludes the QAM that `roles/QAManager/` already claims) | QA1 (`a0241516-…`, paused), QA2 (`bcd265fb-…`) |
| `roles/Dev/` | `role=engineer` | Dev1 (`d7971be8-…`, paused), Dev2 (`6f4fb85e-…`) |

## Hard-fail rules at install time

Both rules MUST be enforced **before any disk write** by the install operation
(and by every future operation that consumes the normalization table). The
rules are part of non-negotiable Q4 (normalize-at-install) and the
matching SKILL.md workflow steps are the single place where the literal
reject-message phrases are specified.

1. **Zero-match rule.** Any methodology source dir resolving to **zero** agents
   in the target company → reject install with the missing dir(s) named.
   Verifies that the target company is structurally complete enough to host
   the methodology before the install touches disk. The literal reject phrase
   is `resolved to zero agents` (see [SKILL.md](../SKILL.md) workflow step 6).

2. **Multi-match rule.** Any single agent matching **two or more** methodology
   source dirs (i.e., the match rules above overlap on that agent) → reject
   install with the conflicting agent and dirs named. Verifies that the
   normalization table is unambiguous on the target company's actual agent
   roster. The literal reject phrase is `matches multiple methodology source
   dirs` (see [SKILL.md](../SKILL.md) workflow step 6).

3. **Single point of update.** This table is the canonical normalization
   surface. If Paperclip-core introduces new enum values (e.g., adds the
   actual `audit` and `qa_manager` enums this table currently treats as
   preferred-but-fall-back), the table is updated here once and every
   operation that imports the table picks up the change.

## Why a separate file

Keeping the table out of the skill body makes the maintenance path explicit
(one file to edit on a Paperclip-core role-enum change) and preserves the
skill body's focus on the operation workflow. The pattern matches the
[gate-review/references/{code-gate.md, disposition-template.md}](/COPAAA/issues/COPAAA-49)
split that Dev2 landed at COPAAA-49 — material that is referenced by but
not part of the operational workflow goes to a sibling reference file.
