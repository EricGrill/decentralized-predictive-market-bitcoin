# Design Review

> Critical questions requiring resolution before implementation

[< Back to Documentation](../README.md)

---

## Overview

This section documents **open design questions** identified during protocol review. Each question targets a specific component where the current specification has gaps, ambiguities, or unexamined trust assumptions.

**Purpose:** Ensure all critical issues are addressed before implementation begins.

---

## Resolution Tracker

| ID | Question | Severity | MVP Blocker | Status | Owner |
|----|----------|----------|-------------|--------|-------|
| Q1 | [ZK Genesis Trust Boundary](Q1-zk-genesis.md) | Critical | No | 🟡 In Progress | - |
| Q2 | [Witness Economic Model](Q2-witness-economics.md) | Critical | No | 🔴 Open | - |
| Q3 | [UTXO Stability](Q3-utxo-stability.md) | High | No | 🔴 Open | - |
| Q4 | [Winner Identification](Q4-winner-identification.md) | Critical | **Yes** | 🔴 Open | - |
| Q5 | [Information Asymmetry](Q5-information-asymmetry.md) | High | No | 🔴 Open | - |

**Legend:**
- 🔴 Open - Unresolved
- 🟡 In Progress - Being worked on
- 🟢 Resolved - Solution accepted

---

## Questions by Component

### Capsule System
- [Q1: ZK Genesis Trust Boundary](Q1-zk-genesis.md)

### Witness Network
- [Q2: Witness Economic Model](Q2-witness-economics.md)

### Escrow System
- [Q3: UTXO Stability](Q3-utxo-stability.md)
- [Q4: Winner Identification](Q4-winner-identification.md)

### Market Design
- [Q5: Information Asymmetry](Q5-information-asymmetry.md)

---

## Question Template

Each question document follows this structure:

```markdown
# Q[N]: [Title]

## Summary
One-paragraph description of the issue.

## Severity
Critical | High | Medium | Low

## MVP Blocker
Yes | No

## The Question
Detailed description of what's unclear or missing.

## Why This Matters
Impact analysis and consequences of not resolving.

## Technical Analysis
Deep dive into the problem, attack vectors, edge cases.

## Proposed Resolutions
Options with trade-offs.

## Recommended Approach
Which option and why.

## Acceptance Criteria
How we know the question is resolved.

## Related
Links to related documents and questions.
```

---

## How to Propose Resolutions

1. **Discuss first**: Open a [Discussion](../../../../discussions) with your proposal
2. **Get feedback**: Gather input from maintainers and community
3. **Submit PR**: Update the question document with accepted resolution
4. **Update tracker**: Mark status as 🟢 Resolved

### Resolution Acceptance Criteria

A resolution is accepted when:
- [ ] Solution addresses the core concern
- [ ] Trade-offs are documented
- [ ] Implementation path is clear
- [ ] At least 2 maintainers approve
- [ ] Spec documents are updated accordingly

---

## Priority Order

Recommended resolution order based on dependencies:

```
1. Q4: Winner Identification (MVP Blocker)
   └─> Must resolve before escrow system can be implemented

2. Q2: Witness Economics
   └─> Must resolve before witness network design is finalized

3. Q1: ZK Genesis
   └─> Can proceed with documented trust assumptions for MVP

4. Q3: UTXO Stability
   └─> Can use K-confirmation rule for MVP

5. Q5: Information Asymmetry
   └─> Can restrict predicates for MVP
```

---

## Summary: Trust Relocation

These questions reveal a common pattern: **trust is relocated, not eliminated**.

| Component | Trust Relocated To |
|-----------|-------------------|
| Key integrity | ZK proof generation environment |
| Market resolution | Witness economic incentives |
| Payout fairness | Off-chain coordination / registry |
| Outcome authenticity | Predicate design constraints |

The protocol is honest about its non-guarantees (see [Security Analysis](../spec/06-security-analysis.md)). These questions inform what additional guarantees may be achievable.
