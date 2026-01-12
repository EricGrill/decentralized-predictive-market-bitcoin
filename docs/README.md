# Protocol Documentation

> Bitcoin-Native Capsule Prediction Market Protocol

[< Back to Repository](../README.md)

---

## Documentation Structure

```
docs/
├── README.md           # This file
├── spec/               # Protocol specification
│   ├── 00-overview.md
│   ├── 01-capsule-model.md
│   ├── 02-bitcoin-anchoring.md
│   ├── 03-market-definition.md
│   ├── 04-escrow-system.md
│   ├── 05-witness-network.md
│   └── 06-security-analysis.md
├── review/             # Design questions and analysis
│   ├── README.md
│   ├── Q1-zk-genesis.md
│   ├── Q2-witness-economics.md
│   ├── Q3-utxo-stability.md
│   ├── Q4-winner-identification.md
│   └── Q5-information-asymmetry.md
└── appendix/           # Technical appendices
    ├── serialization.md
    ├── taproot-scripts.md
    └── frost-protocol.md
```

---

## Protocol Specification

Start here for understanding how the protocol works.

| Document | Description | Status |
|----------|-------------|--------|
| [Overview](spec/00-overview.md) | Goals, components, architecture | Draft |
| [Capsule Model](spec/01-capsule-model.md) | Capsule structure, ZK proofs | Draft |
| [Bitcoin Anchoring](spec/02-bitcoin-anchoring.md) | OP_RETURN commitments | Draft |
| [Market Definition](spec/03-market-definition.md) | Markets, predicates, fees | Draft |
| [Escrow System](spec/04-escrow-system.md) | Taproot escrows, payouts | Draft |
| [Witness Network](spec/05-witness-network.md) | Attestation, bonding, slashing | Draft |
| [Security Analysis](spec/06-security-analysis.md) | Failure modes, trust model | Draft |

**Reading order:** 00 → 01 → 02 → 03 → 04 → 05 → 06

---

## Design Review

Critical questions that must be resolved before implementation.

| Question | Topic | Severity | Status |
|----------|-------|----------|--------|
| [Q1](review/Q1-zk-genesis.md) | ZK Genesis Proof Trust | Critical | Open |
| [Q2](review/Q2-witness-economics.md) | Witness Economic Model | Critical | Open |
| [Q3](review/Q3-utxo-stability.md) | UTXO Stability | High | Open |
| [Q4](review/Q4-winner-identification.md) | Winner Identification | **Critical (MVP Blocker)** | Open |
| [Q5](review/Q5-information-asymmetry.md) | Information Asymmetry | High | Open |

See [Review Index](review/README.md) for full details.

---

## Technical Appendices

Reference material for implementers.

| Appendix | Description | Status |
|----------|-------------|--------|
| [Serialization](appendix/serialization.md) | Canonical encodings, test vectors | Stub |
| [Taproot Scripts](appendix/taproot-scripts.md) | Full script templates | Stub |
| [FROST Protocol](appendix/frost-protocol.md) | Threshold signing details | Stub |

---

## Key Concepts

### Trust Minimization

The protocol relocates trust rather than eliminating it:

| Layer | Trust Assumption | Mitigation |
|-------|------------------|------------|
| Bitcoin | 51% honest hashrate | 15+ year track record |
| Capsules | ZK proof soundness | Hardware attestation |
| Witnesses | ≥T honest of N | Economic bonding + slashing |
| Predicates | Unambiguous specification | Formal spec + test vectors |
| Payouts | Correct winner list | **(Unspecified - Critical gap)** |

### BTC-Only Design Principles

1. **Denomination**: All stakes and payouts in BTC
2. **Settlement**: Bitcoin transactions only
3. **Anchoring**: OP_RETURN commitments (no UTXO pollution)
4. **Escrow**: Taproot with conditional spend paths
5. **Time**: Bitcoin block height as universal clock

### Deterministic Predicates

Markets resolve based on machine-checkable facts:

```
OP_RETURN_EXISTS(commitment, beforeHeight)   # Did OP_RETURN appear?
TX_CONFIRMED(txid, beforeHeight)             # Is TX confirmed?
ADDRESS_BALANCE_GE(addr, sats, atHeight)     # Balance check
UTXO_EXISTS(outpoint, atHeight)              # UTXO existence
```

No human interpretation. No discretionary oracles.

---

## Version History

See [CHANGELOG.md](../CHANGELOG.md) for full history.

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0-draft | 2026-01-12 | Initial specification, design review |

---

## Contributing to Documentation

1. **Spec improvements**: Open PR with clear rationale
2. **Question resolutions**: Propose in [Discussions](../../../discussions) first
3. **Test vectors**: Add to appendix with verification steps
4. **Typos/clarity**: Direct PRs welcome

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.
