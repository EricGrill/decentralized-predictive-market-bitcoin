# Bitcoin-Native Capsule Prediction Market Protocol

> BTC-Denominated, BTC-Settled Markets for Verifiable Cryptographic Outcomes

[![Protocol Status](https://img.shields.io/badge/status-research-yellow)](docs/README.md)
[![Version](https://img.shields.io/badge/version-0.1.0--draft-blue)](CHANGELOG.md)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## Overview

A Bitcoin-only prediction market protocol where markets resolve on the future state of cryptographic "capsules" anchored to Bitcoin. The system enables:

- **Auditable markets** without disclosure (capsules expose public keys, keep private keys encrypted)
- **Deterministic resolution** via machine-checkable predicates over Bitcoin-observable facts
- **Decentralized attestation** through threshold Schnorr signatures from bonded witnesses
- **BTC-only settlement** using Taproot escrow and payout paths

No smart-contract platforms. No stablecoins. No external oracles.

---

## Quick Links

| Documentation | Description |
|---------------|-------------|
| [Protocol Specification](docs/README.md) | Full protocol documentation |
| [Design Review](docs/review/README.md) | Open questions and technical analysis |
| [Original Paper](paper.md) | Canonical protocol specification |
| [Changelog](CHANGELOG.md) | Version history |
| [Contributing](CONTRIBUTING.md) | How to contribute |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BITCOIN BLOCKCHAIN                            │
│     OP_RETURN Anchors  │  Taproot Escrows  │  Witness Bonds         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       PROTOCOL LAYER                                 │
│   Capsules ──── Markets ──── Predicates ──── Witnesses              │
└─────────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Purpose | Documentation |
|-----------|---------|---------------|
| **Capsules** | Cryptographic containers with verifiable key origin | [Spec](docs/spec/01-capsule-model.md) |
| **Bitcoin Anchoring** | OP_RETURN commitments for timestamping | [Spec](docs/spec/02-bitcoin-anchoring.md) |
| **Markets** | YES/NO bets on capsule state predicates | [Spec](docs/spec/03-market-definition.md) |
| **Escrow System** | Taproot-based conditional payouts | [Spec](docs/spec/04-escrow-system.md) |
| **Witness Network** | Decentralized oracle with threshold signing | [Spec](docs/spec/05-witness-network.md) |

---

## Status

**Current Phase:** Research & Specification

| Milestone | Status |
|-----------|--------|
| Protocol specification | In Progress |
| Design review | In Progress |
| Reference implementation | Not Started |
| Testnet deployment | Not Started |
| Security audit | Not Started |
| Mainnet | Not Started |

### Open Design Questions

Five critical questions require resolution before implementation:

1. [Q1: ZK Genesis Proof Trust Boundary](docs/review/Q1-zk-genesis.md) - How do we verify proofs came from constrained environments?
2. [Q2: Witness Economic Model](docs/review/Q2-witness-economics.md) - What incentivizes witness participation?
3. [Q3: UTXO Stability](docs/review/Q3-utxo-stability.md) - How do we handle reorgs affecting escrow deposits?
4. [Q4: Winner Identification](docs/review/Q4-winner-identification.md) - **Critical MVP blocker** - How do we track who staked what?
5. [Q5: Information Asymmetry](docs/review/Q5-information-asymmetry.md) - How do we prevent capsule holders from manipulating outcomes?

See [Resolution Tracker](docs/review/README.md#resolution-tracker) for current status.

---

## Related Projects

| Repository | Description |
|------------|-------------|
| [mcp-memvid-state-service](https://github.com/EricGrill/mcp-memvid-state-service) | Memvid state service MCP |
| [mcp-predictive-market](https://github.com/EricGrill/mcp-predictive-market) | Predictive market MCP |
| [mcp-bitcoin-cli](https://github.com/EricGrill/mcp-bitcoin-cli) | Bitcoin CLI MCP |

---

## Getting Started

### For Researchers

1. Read the [Protocol Overview](docs/spec/00-overview.md)
2. Review [Open Questions](docs/review/README.md)
3. Open a [Discussion](../../discussions) to propose solutions

### For Implementers

1. Read the full [Specification](docs/README.md)
2. Review [Security Analysis](docs/spec/06-security-analysis.md)
3. Check [Implementation Checklist](docs/spec/06-security-analysis.md#implementation-security-checklist)

### For Auditors

1. Review [Trust Model](docs/spec/06-security-analysis.md#trust-model-summary)
2. Examine [Attack Cost Analysis](docs/spec/06-security-analysis.md#attack-cost-analysis)
3. Check [Open Questions](docs/review/README.md) for known gaps

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Ways to contribute:**
- Propose solutions to [open design questions](docs/review/README.md)
- Improve specification clarity
- Add test vectors
- Review for security issues
- Build reference implementations

---

## License

MIT License - See [LICENSE](LICENSE) for details.

---

## Acknowledgments

Built on Bitcoin. Inspired by the need for trustless, BTC-native prediction markets.
