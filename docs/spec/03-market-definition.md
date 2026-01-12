# Market Definition

> Deterministic predicates and market structure

[< Back to Bitcoin Anchoring](02-bitcoin-anchoring.md) | [Next: Escrow System >](04-escrow-system.md)

---

## Overview

A market is a Bitcoin-anchored commitment to a future **yes/no question** about a capsule's state.

---

## MarketDefinition Structure

```
MarketDefinition {
  version: u8
  capsuleCIDCommit: bytes32      // Commitment to capsule CID (hash)
  predicateType: u8
  predicateParams: bytes
  resolutionHeight: u32          // Bitcoin block height
  witnessSetHash: bytes32        // Commitment to witness set
  threshold: u8                  // T of N
  payoutRule: u8                 // e.g., pro-rata
  feePolicy: bytes
}
```

### Field Descriptions

| Field | Description |
|-------|-------------|
| `version` | Protocol version for forward compatibility |
| `capsuleCIDCommit` | SHA256 of capsule CID being bet on |
| `predicateType` | Type code for predicate (see below) |
| `predicateParams` | TLV-encoded predicate parameters |
| `resolutionHeight` | Block height at which market resolves |
| `witnessSetHash` | Which witness set will resolve this market |
| `threshold` | T signatures required from N witnesses |
| `payoutRule` | How winnings are distributed |
| `feePolicy` | Fee structure (witness fees, protocol fees) |

---

## Market Commitment

```
marketHash = SHA256(MarketDefinition)
```

The market is created by broadcasting a Bitcoin transaction:

```
OP_RETURN <0x01 | 0x02 | marketHash>
```

This timestamps the market and makes its rules **immutable**.

---

## Predicate System

Predicates MUST be:
- **Deterministic**: Same inputs always produce same output
- **Bitcoin-verifiable**: Objectively checkable using Bitcoin data
- **Binary**: Returns YES (1) or NO (0)

### Allowed Predicate Types

| Type Code | Predicate | Description |
|-----------|-----------|-------------|
| `0x01` | `OP_RETURN_EXISTS(commitment, beforeHeight)` | Check if OP_RETURN with commitment exists |
| `0x02` | `TX_CONFIRMED(txid, beforeHeight)` | Check if transaction is confirmed |
| `0x03` | `ADDRESS_BALANCE_GE(scriptPubKey, sats, atHeight)` | Check address balance ≥ threshold |
| `0x04` | `UTXO_EXISTS(outpoint, atHeight)` | Check if specific UTXO exists |
| `0x05` | `SUCCESSOR_CAPSULE_ANCHORED(successorCommit, beforeHeight)` | Check for capsule succession |

### Predicate Encoding

`predicateParams` uses canonical TLV (Tag-Length-Value) encoding:

```
┌───────┬────────┬─────────┐
│ Tag   │ Length │ Value   │
│ 1 byte│ 2 bytes│ N bytes │
└───────┴────────┴─────────┘
```

**Rules:**
- All encodings MUST be canonical (unique serialization)
- Parameters MUST include any commitments needed to avoid long identifiers
- Multi-value predicates chain TLV entries

### Example: OP_RETURN_EXISTS

```
predicateType: 0x01
predicateParams:
  ┌─────────────────────────────────────────────┐
  │ Tag: 0x01 (commitment)                      │
  │ Length: 0x0020 (32 bytes)                   │
  │ Value: <32-byte commitment hash>            │
  ├─────────────────────────────────────────────┤
  │ Tag: 0x02 (beforeHeight)                    │
  │ Length: 0x0004 (4 bytes)                    │
  │ Value: <block height as u32 big-endian>     │
  └─────────────────────────────────────────────┘
```

---

## Payout Rules

| Code | Rule | Description |
|------|------|-------------|
| `0x01` | Pro-rata | Winners split proportionally to stake |
| `0x02` | Winner-take-all | Largest stake wins all |
| `0x03` | Lottery | Random winner weighted by stake |

### Pro-rata (MVP Default)

```
winner_payout[i] = (stake[i] / total_winning_stake) * total_escrow
```

**Implementation note:** Requires tracking who staked what.

> See [Q4: Winner Identification](../review/Q4-winner-identification.md) for open questions

---

## Fee Policy

```
FeePolicy {
  version: u8
  feeType: u8                    // Fixed, proportional, or hybrid
  feeParams: bytes               // Type-specific parameters
  witnessAllocationBPS: u16      // Basis points to witness pool
  protocolAllocationBPS: u16     // Basis points to protocol treasury
  burnAllocationBPS: u16         // Basis points burned
}
```

### Fee Types

| Type | Description | Params |
|------|-------------|--------|
| `0x01` | Fixed sats | `amount: u64` |
| `0x02` | Proportional BPS | `bps: u16` (1% = 100) |
| `0x03` | Hybrid | `fixed: u64, bps: u16` |

### Example: 1% Fee, 70/20/10 Split

```
FeePolicy {
  version: 1
  feeType: 0x02                  // Proportional
  feeParams: 0x0064              // 100 BPS = 1%
  witnessAllocationBPS: 7000     // 70%
  protocolAllocationBPS: 2000    // 20%
  burnAllocationBPS: 1000        // 10%
}
```

---

## Market Lifecycle

```
1. CREATION
   └─> MarketDefinition built and anchored
   └─> YES/NO escrow addresses generated

2. STAKING (before resolutionHeight)
   └─> Participants send BTC to escrow addresses
   └─> Stakes are tracked (implementation-dependent)

3. RESOLUTION (at resolutionHeight + K confirmations)
   └─> Witnesses evaluate predicate
   └─> Threshold signature produced for outcome

4. PAYOUT
   └─> Winning escrow spent via script path
   └─> BTC distributed per payoutRule
   └─> Fees distributed per feePolicy
```

---

## Related Documents

- [Escrow System](04-escrow-system.md) - How stakes are held and paid out
- [Witness Network](05-witness-network.md) - Who resolves the predicate
- [Q2: Witness Economics](../review/Q2-witness-economics.md) - Fee incentive analysis
