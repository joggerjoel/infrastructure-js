# Architecture Decision Records (ADR)

---
**Status**: 💡 **GUIDANCE** - Template and process for documenting architectural decisions  
**Last Updated**: 2026-01-28  
**Owned By**: Architecture Team  

---

## Overview

Architecture Decision Records (ADRs) capture the **why** behind architectural decisions, not just the **what**. They prevent tribal knowledge loss and answer "why the hell did we do this?" six months later.

## ADR Process

1. **Propose**: Create ADR with status "Proposed"
2. **Review**: Team reviews, discusses alternatives
3. **Decide**: Update status to "Accepted" or "Rejected"
4. **Implement**: Reference ADR in implementation PRs
5. **Deprecate**: If superseded, mark as "Deprecated" and link to new ADR

## ADR Status

- **Proposed**: Under discussion, not yet decided
- **Accepted**: Decision made, implementation approved
- **Deprecated**: Superseded by another ADR
- **Rejected**: Decision was not made, alternative chosen

## Naming Convention

ADRs are numbered sequentially: `0001-short-title.md`, `0002-another-decision.md`, etc.

## Template

See [template.md](./template.md) for the ADR template.

## Index

<!-- ADRs will be listed here as they are created -->

---

**Last Updated**: 2026-01-28  
**Next Review**: 2026-04-28 (Quarterly)
