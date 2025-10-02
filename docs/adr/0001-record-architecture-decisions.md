# 0001 – Record Architecture Decisions (ADRs)
**Status:** Accepted  
**Deciders:** Umashankar Ankuri, John Smith, Alex Johnson  
**Date:** 2025-09-28

## Context and Problem Statement
As the food delivery platform evolves, we will make numerous architectural choices (e.g., service boundaries, messaging, persistence, mobile tech). Without a lightweight, consistent way to capture these decisions, context gets lost, onboarding is slower, and decisions are repeatedly debated. We need a simple, versioned mechanism to document **what we decided, why, and the trade‑offs** at the time.

## Decision Drivers
- **Traceability:** Make the rationale discoverable alongside code and docs.
- **Speed:** Enable quick, reviewable decisions without heavyweight processes.
- **Consistency:** Standardize the template and storage location.
- **Accountability:** Capture deciders and dates for future audits and handovers.
- **Change management:** Allow decisions to be superseded as the system scales.

## Considered Options
1. **No formal decision log** (rely on PRs/meetings)  
2. **Wiki/Confluence pages** (unversioned, hard to diff)  
3. **Markdown ADRs in-repo** (numbered, reviewable, versioned) — **Chosen**

## Decision Outcome
**Chosen option:** *Markdown ADRs in-repo* following a simple template (this one).  
- **Location:** `/adr` at the repo root.  
- **Format:** `NNNN-title-with-dashes.md` (zero‑padded sequence).  
- **Lifecycle:** *Proposed → Accepted → Deprecated → Superseded*.  
- **Indexing:** Link the ADR folder from `README.md`.  
- **Cross‑references:** Each ADR links to related ADRs and docs.

## Positive Consequences
- Clear, versioned history of architectural reasoning.
- Faster onboarding; fewer repeated debates.
- Easier audits and compliance reviews.
- Decisions are easy to diff/review via standard PR flow.

## Negative Consequences
- Light maintenance overhead to write/curate ADRs.
- Requires discipline to update status (e.g., when superseded).
- Might feel duplicative with PR descriptions if not cross‑linked.

## Follow‑ups and Conventions
- Add an **ADR index** to `README.md` → *“See /adr for decisions and status.”*
- Encourage referencing ADR IDs in PR titles/descriptions (e.g., `ADR-0004`).
- When a decision changes, create a **new ADR** and mark the old one **Superseded by NNNN**.

