# RFC: [Nome da Proposta]

| Field | Value |
|-------|-------|
| **Author(s)** | [names] |
| **Approver(s)** | [names — who needs to sign off] |
| **Status** | Draft · In Review · Approved · Superseded · Deprecated |
| **Created** | [date] |
| **Last Updated** | [date] |
| **Team** | [owning team] |

---

## Abstract

[3-5 sentence executive summary. State what the RFC proposes, why it matters,
and the key trade-off or insight that makes this proposal non-obvious.
A reader should be able to decide whether to read the full RFC from this alone.]

---

## Motivation

[Why are we doing this? Ground in concrete data or incidents —
"Users experience 3s load times on the dashboard" not "The page is slow."
Include links to metrics, incident reports, or user feedback.]

---

## Goals and Non-Goals

**Goals:**
- [Specific, measurable. "P99 latency under 200ms" not "fast."]
- [Each goal should be verifiable after the project ships.]

**Non-Goals:**
- [What this project will NOT do. Prevents scope creep.]
- [If a non-goal is planned for later, say so.]

---

## Approaches

Present all evaluated options together. Each approach should be given a fair treatment with genuine pros and cons.

### Approach 1: [Name]

**Description:** [How this approach works at a high level.]

**Architecture:**

```
    +------------------+     +------------------+     +------------------+
    |   Component A    |---->|   Component B    |---->|   Component C    |
    |   (description)  |     |   (description)  |     |   (description)  |
    +------------------+     +------------------+     +------------------+
```

**Pros:**
- [Genuine advantage]
- [Genuine advantage]

**Cons:**
- [Honest trade-off]
- [Honest trade-off]

### Approach 2: [Name]

[Same structure as Approach 1.]

### Approach 3: [Name] (if applicable)

[Same structure.]

### Recommendation

**Chosen approach:** [Name]

**Justification:** [Why this approach wins given the specific goals, constraints,
and trade-offs. Acknowledge what you're giving up relative to the alternatives.]

---

## Open Questions

1. [Question — include context and your leaning]
2. [Question]

---

## References

- [Related RFCs, ADRs, prior art, dashboards, incident reports]