# Appendix: FROST Threshold Signing Protocol

> Threshold Schnorr signatures for witness network

[< Back to Documentation](../README.md)

---

## Status

**Stub** - To be completed during implementation phase.

---

## Overview

FROST (Flexible Round-Optimized Schnorr Threshold) enables T-of-N threshold signatures where:
- N participants hold key shares
- Any T participants can produce a valid signature
- Signature is indistinguishable from single-signer Schnorr

---

## Why FROST?

| Property | Requirement | FROST |
|----------|-------------|-------|
| Threshold | T-of-N signing | Yes |
| Schnorr compatible | BIP340 verification | Yes |
| Round efficient | Minimize coordination | 2 rounds |
| Robust | Handles malicious participants | Yes (with identifiable abort) |

---

## Protocol Phases

### Key Generation (DKG)

```
TODO: Document distributed key generation
- Feldman VSS or Pedersen DKG
- Public verification of shares
- Aggregate public key derivation
```

### Signing Round 1: Commitment

```
TODO: Document nonce commitment phase
```

### Signing Round 2: Signature Share

```
TODO: Document signature share generation
```

### Aggregation

```
TODO: Document signature aggregation
```

---

## Implementation Options

### Libraries

| Library | Language | Status |
|---------|----------|--------|
| [frost-secp256k1](https://github.com/ZcashFoundation/frost) | Rust | Production |
| [secp256k1-zkp](https://github.com/ElementsProject/secp256k1-zkp) | C | Production |

### Considerations

- Must use secp256k1 curve (Bitcoin compatible)
- Must produce BIP340-compatible signatures
- Must handle participant failures gracefully

---

## Witness Network Integration

```
TODO: Document:
- How witnesses coordinate signing rounds
- Message format for partial signatures
- Aggregator role and trust model
- Timeout and retry handling
```

---

## Security Analysis

```
TODO: Document:
- Security assumptions
- Attack resistance (rogue key, etc.)
- Comparison with MuSig2
```

---

## Test Vectors

```
TODO: Add test vectors for:
- Key generation
- Signing rounds
- Aggregation
- Invalid share detection
```

---

## Related

- [Spec: Witness Network](../spec/05-witness-network.md)
- [FROST Paper](https://eprint.iacr.org/2020/852)
- [BIP 340: Schnorr Signatures](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
