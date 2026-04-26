# CopperForge Methodology

This directory is the canonical, versioned home of the CopperForge methodology — the role docs, workflow rules, gates, and standards that every Paperclip company running on CopperForge inherits.

## Layout

```
methodology/
├── COMPANY.md           CopperPress identity and core principles
├── PLATFORM.md          The methodology — workflow, gates, pipelining, escalation
├── CODING_STANDARDS.md  Technical standards enforced at Code Gate
└── roles/
    ├── CEO/             AGENTS, SOUL, HEARTBEAT, TOOLS
    ├── CTO/             AGENTS, SOUL, HEARTBEAT
    ├── BA/              AGENTS
    ├── Auditor/         AGENTS
    ├── QAManager/       AGENTS
    ├── QA/              AGENTS
    └── Dev/             AGENTS
```

CEO has the full four-file split (judgment-heavy role). CTO has three (tools folded into AGENTS). The other five roles are narrow enough that everything lives in a single AGENTS.md.

## How companies use this

A Paperclip company stands up CopperForge by registering against a specific tagged version of this repository. The methodology files are placed in the company's config directory by the `methodology-bootstrap` skill (see the skills directory).

Companies do not edit these files directly. Methodology changes happen in this directory, are reviewed and merged via PR, and are released as new tagged versions. Companies upgrade by bumping the version they track.

## Versioning policy

Methodology releases follow semantic versioning:

- **Major version bump** (1.x.x → 2.0.0) — incompatible methodology changes (gate restructure, role removal, fundamental flow change). Companies must opt in.
- **Minor version bump** (1.0.x → 1.1.0) — additive changes (new role, new gate, new principle). Companies upgrade at their cadence.
- **Patch version bump** (1.0.0 → 1.0.1) — clarifications, typos, non-substantive edits.

## Editing this methodology

Methodology changes require board approval. The "no agent modifies their own role docs" rule extends to all founding configuration in this directory. Agents propose; board approves; PR merges.
