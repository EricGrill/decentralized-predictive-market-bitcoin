# Appendix: Canonical Serialization

> Encoding formats and test vectors

[< Back to Documentation](../README.md)

---

## Status

**Stub** - To be completed during implementation phase.

---

## Overview

All protocol objects must have canonical (unique) serializations to ensure:
- Deterministic hash computation
- Cross-implementation compatibility
- Unambiguous commitment verification

---

## Encoding Rules

### General Principles

1. **Integers**: Big-endian, fixed width (u8, u16, u32, u64)
2. **Bytes**: Length-prefixed with u16 length
3. **Strings**: UTF-8, length-prefixed with u16
4. **Optional fields**: 1-byte presence flag (0x00 = absent, 0x01 = present)
5. **Arrays**: u16 count prefix, followed by elements

### TLV (Tag-Length-Value)

For variable structures:

```
┌───────┬────────┬─────────┐
│ Tag   │ Length │ Value   │
│ 1 byte│ 2 bytes│ N bytes │
└───────┴────────┴─────────┘
```

---

## Object Encodings

### Capsule

```
TODO: Define canonical encoding
```

### MarketDefinition

```
TODO: Define canonical encoding
```

### WitnessSet

```
TODO: Define canonical encoding
```

### OutcomeMessage

```
TODO: Define canonical encoding
```

### FeePolicy

```
TODO: Define canonical encoding
```

---

## Test Vectors

### MarketDefinition

```
TODO: Add test vectors with:
- Input object (JSON for readability)
- Canonical bytes (hex)
- SHA256 hash
```

### OutcomeMessage

```
TODO: Add test vectors
```

---

## Implementation Notes

- Use existing libraries where possible (e.g., Bitcoin's serialization)
- Test cross-implementation compatibility early
- Version field allows future format changes

---

## Related

- [Spec: Market Definition](../spec/03-market-definition.md)
- [Spec: Bitcoin Anchoring](../spec/02-bitcoin-anchoring.md)
