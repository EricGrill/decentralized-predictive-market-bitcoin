# Protocol Overview

> Bitcoin-Native Capsule Prediction Market Protocol

[< Back to Index](../README.md) | [Next: Capsule Model >](01-capsule-model.md)

---

## Abstract

This document specifies a **Bitcoin-only** predictive market protocol whose markets resolve on the future state of cryptographic "capsules" that are anchored to Bitcoin. The system is designed to be auditable without disclosure: capsules expose Bitcoin public keys for verification while keeping private keys encrypted, and markets are denominated and settled entirely in BTC.

Because Bitcoin Script cannot introspect Bitcoin's own chain state, markets require a **decentralized witness network** (a constrained, replaceable attestation layer) that evaluates deterministic predicates over Bitcoin-observable facts and produces **threshold Schnorr signatures** to unlock Taproot escrow paths.

The protocol avoids smart-contract platforms, stablecoins, and external oracle networks. It relies on Bitcoin for time and finality; on Taproot for enforceable escrow and payout paths; on deterministic predicate definitions for unambiguous resolution; and on cryptographic origin guarantees using **zero-knowledge proofs of sealed key genesis** to prevent "tainted capsule" attacks.

---

## Goals

| Goal | Description |
|------|-------------|
| **BTC-only denomination** | Staking and payouts in BTC only |
| **Public verifiability** | Anyone can verify capsule-linked Bitcoin state |
| **Deterministic resolution** | Markets resolve from unambiguous, machine-checkable predicates |
| **No discretionary oracles** | Witnesses attest facts, never interpret meaning |
| **No UTXO pollution** | Commitments use OP_RETURN; escrow uses Taproot |
| **Tainted-origin prevention** | Capsules MUST include ZK proofs of sealed key genesis |
| **Replaceable components** | Witness sets are market-scoped and upgradeable |

## Non-Goals

- Retail wallet UX
- Anonymity guarantees against blockchain analysis
- On-chain computation or Turing-complete settlement
- Stablecoin pegs or cross-chain bridges
- High-frequency trading or instant resolution

---

## System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BITCOIN BLOCKCHAIN                            │
│  ┌───────────────┐  ┌──────────────┐  ┌────────────────────────┐   │
│  │ OP_RETURN     │  │ Taproot      │  │ Witness Bond           │   │
│  │ Commitments   │  │ Escrow UTXOs │  │ UTXOs                  │   │
│  └───────────────┘  └──────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       PROTOCOL LAYER                                 │
│                                                                      │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────────────┐   │
│  │  Capsules   │   │   Markets    │   │   Witness Network      │   │
│  │             │   │              │   │                        │   │
│  │ • CID       │   │ • Predicate  │   │ • N nodes, T threshold │   │
│  │ • pubkey    │   │ • Escrow     │   │ • FROST/MuSig2         │   │
│  │ • ZK proof  │   │ • Witness Set│   │ • BTC-bonded           │   │
│  └─────────────┘   └──────────────┘   └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Summary

1. **Capsules**: Structured objects exposing a Bitcoin public key with encrypted private key and proofs
2. **Bitcoin Anchors**: OP_RETURN commitments for timestamping capsules and market definitions
3. **Prediction Markets**: BTC-staked YES/NO markets bound to capsules and deterministic predicates
4. **Taproot Escrows**: Enforceable conditional payout paths for market outcomes
5. **Witness Network**: Bitcoin-native attestors using threshold Schnorr signatures
6. **Slashing & Governance**: Bitcoin-native bonding, penalties, and witness-set updates

---

## Summary

| Component | Provides |
|-----------|----------|
| Bitcoin | Time (block height), truth (confirmed txs/UTXOs), finality |
| Taproot | Enforceable escrow and conditional payout paths |
| Witnesses | Decentralized attestation of deterministic predicates |
| Capsules | Verifiable objects with provable clean key origin |

This protocol turns capsule evolution into BTC-denominated prediction markets resolved by Bitcoin facts and enforced by Bitcoin spending conditions.

---

## Related Documents

- [Capsule Model](01-capsule-model.md) - Capsule structure and ZK genesis proofs
- [Bitcoin Anchoring](02-bitcoin-anchoring.md) - OP_RETURN commitment system
- [Market Definition](03-market-definition.md) - Market structure and predicates
- [Escrow System](04-escrow-system.md) - Taproot escrows and payouts
- [Witness Network](05-witness-network.md) - Attestation, bonding, slashing
- [Security Analysis](06-security-analysis.md) - Failure modes and guarantees

---

## References

**MCP Repositories (context):**
- Memvid State Service: https://github.com/EricGrill/mcp-memvid-state-service
- Predictive Market MCP: https://github.com/EricGrill/mcp-predictive-market
- Bitcoin CLI MCP: https://github.com/EricGrill/mcp-bitcoin-cli
