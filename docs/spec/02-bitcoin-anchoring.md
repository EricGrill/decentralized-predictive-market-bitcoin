# Bitcoin Anchoring

> Immutable timestamping via OP_RETURN commitments

[< Back to Capsule Model](01-capsule-model.md) | [Next: Market Definition >](03-market-definition.md)

---

## OP_RETURN Definition

OP_RETURN is a Bitcoin Script opcode used to create **provably unspendable outputs** that can carry a small data payload, without bloating the UTXO set.

This protocol uses OP_RETURN strictly for **commitments** (hashes), not content.

```
OP_RETURN <commitment_data>
```

---

## Anchor Types

| Anchor Type | Commits To | Purpose |
|-------------|------------|---------|
| **Capsule Anchor** | Capsule CID (or hash) | Timestamp capsule existence |
| **Market Definition Anchor** | `marketHash` | Make market rules immutable |
| **Witness Set Anchor** | `witnessSetHash` | Track witness configuration changes |

---

## Payload Format

### Size Constraint

OP_RETURN payload must fit standard relay policies: **commonly 80 bytes**.

CIDs or definitions longer than this MUST be compacted using a hash commitment.

### Commitment Structure

```
OP_RETURN payload (≤80 bytes):

┌──────────┬─────────────┬──────────────────────────┐
│ Version  │ Anchor Type │ Commitment Hash          │
│ (1 byte) │ (1 byte)    │ (32 bytes = SHA256)      │
└──────────┴─────────────┴──────────────────────────┘

Total: 34 bytes (fits comfortably in 80-byte limit)
```

### Anchor Type Codes

```
0x01 = CAPSULE_ANCHOR
0x02 = MARKET_DEFINITION_ANCHOR
0x03 = WITNESS_SET_ANCHOR
0x04 = REDEMPTION_ANCHOR (for capsule lifecycle events)
```

---

## Anchoring Workflow

### Capsule Anchor

```
1. Build capsule object (pubkey, enc_privkey, zk_proof, ...)
2. Compute capsuleCID = CID(Capsule)
3. Compute commitment = SHA256(capsuleCID)
4. Broadcast Bitcoin TX:
   - Input: funding UTXO
   - Output 0: OP_RETURN <0x01 | 0x01 | commitment>
   - Output 1: change (if any)
5. Capsule is anchored at confirmation height
```

### Market Definition Anchor

```
1. Build MarketDefinition
2. Compute marketHash = SHA256(MarketDefinition)
3. Broadcast Bitcoin TX:
   - Output 0: OP_RETURN <0x01 | 0x02 | marketHash>
4. Market rules are immutable after confirmation
```

### Witness Set Anchor

```
1. Build WitnessSet configuration
2. Compute witnessSetHash = SHA256(WitnessSet)
3. Broadcast Bitcoin TX:
   - Output 0: OP_RETURN <0x01 | 0x03 | witnessSetHash>
4. Witness set change is recorded (subject to activation delay)
```

---

## Verification

To verify an anchor:

1. Locate Bitcoin TX by txid
2. Parse OP_RETURN output
3. Extract commitment hash
4. Obtain full object (capsule/market/witness set) from off-chain storage
5. Verify: `SHA256(object) == commitment`
6. Anchor height = TX confirmation height

---

## Design Rationale

### Why Commitments, Not Content?

| Approach | Pros | Cons |
|----------|------|------|
| Full content in OP_RETURN | Self-contained | Size limited to 80 bytes |
| Hash commitment | Unlimited object size | Requires off-chain storage |

This protocol uses **hash commitments** because:
- Market definitions and capsules exceed 80 bytes
- Off-chain storage (IPFS, Arweave, Nostr) provides content availability
- Bitcoin provides immutable timestamp and ordering

### Why OP_RETURN, Not Other Methods?

| Method | UTXO Impact | Standard Relay | Cost |
|--------|-------------|----------------|------|
| OP_RETURN | None (prunable) | Yes | Low |
| Inscription (Ordinals) | Creates UTXO | Varies | Higher |
| P2SH/P2WSH data embedding | Creates UTXO | No | Medium |

OP_RETURN is the cleanest method that doesn't pollute the UTXO set.

---

## Off-Chain Storage Requirements

Objects referenced by anchors MUST be retrievable. Recommended storage:

| Storage | Availability | Cost | Immutability |
|---------|--------------|------|--------------|
| IPFS + Pinning | High (if pinned) | Low | CID-addressed |
| Arweave | Permanent | Medium | Blockchain-backed |
| Nostr | Distributed | Free | Signature-backed |

The protocol does not mandate a specific storage layer, but implementations SHOULD support multiple backends.

---

## Related Documents

- [Market Definition](03-market-definition.md) - What gets anchored
- [Witness Network](05-witness-network.md) - Witness set anchoring
- [Appendix: Serialization](../appendix/serialization.md) - Canonical encoding
