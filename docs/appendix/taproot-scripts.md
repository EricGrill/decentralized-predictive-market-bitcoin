# Appendix: Taproot Script Templates

> Full script templates for escrow construction

[< Back to Documentation](../README.md)

---

## Status

**Stub** - To be completed during implementation phase.

---

## Overview

Each market escrow uses Taproot outputs with:
- Key path for cooperative spending
- Script paths for YES/NO outcomes
- Optional timeout path for refunds

---

## Escrow Structure

```
Taproot Output
│
├── Internal Key (P)
│   └── MuSig2 aggregate of [market_operator, participant_aggregate]
│
└── Script Tree (Merkle Root)
    ├── Leaf 0: YES Outcome Script
    ├── Leaf 1: NO Outcome Script
    └── Leaf 2: Timeout Refund Script (optional)
```

---

## Script Templates

### YES Outcome Script

```
TODO: Define exact script
Pseudocode:
  <witness_aggregate_pk> OP_CHECKSIG

Sighash: covers OutcomeMessage(marketHash, YES, resolutionHeight)
```

### NO Outcome Script

```
TODO: Define exact script
```

### Timeout Refund Script

```
TODO: Define exact script
Pseudocode:
  <timeout_height> OP_CHECKLOCKTIMEVERIFY OP_DROP
  <refund_key> OP_CHECKSIG
```

---

## Address Generation

```rust
// TODO: Implement
fn generate_escrow_addresses(market: &Market) -> (Address, Address) {
    // Returns (YES_escrow, NO_escrow)
}
```

---

## Spending Transactions

### Outcome Spend

```
TODO: Define transaction template for spending via outcome path
```

### Timeout Spend

```
TODO: Define transaction template for refund path
```

---

## Test Vectors

```
TODO: Add test vectors with:
- Market parameters
- Generated addresses
- Valid spending transactions
```

---

## Related

- [Spec: Escrow System](../spec/04-escrow-system.md)
- [BIP 341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)
- [BIP 342: Tapscript](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)
