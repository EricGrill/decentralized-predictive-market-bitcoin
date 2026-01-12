# Bitcoin Anchoring

> Immutable timestamping via OP_RETURN data embedding

[< Back to Capsule Model](01-capsule-model.md) | [Next: Market Definition >](03-market-definition.md)

---

## OP_RETURN Definition

OP_RETURN is a Bitcoin Script opcode used to create **provably unspendable outputs** that can carry arbitrary data payloads without bloating the UTXO set.

```
OP_RETURN <data_payload>
```

---

## Bitcoin Core v30: OP_RETURN Limit Removed

**As of Bitcoin Core v30, the previous 80-byte OP_RETURN standardness limit has been removed.**

This is a significant change for data-anchoring protocols:

| Version | OP_RETURN Limit | Implication |
|---------|-----------------|-------------|
| < v30 | 80 bytes | Must use hash commitments for most data |
| ≥ v30 | **No standardness limit** | Can embed full objects directly |

This protocol leverages the unlimited OP_RETURN to enable **self-contained anchors** where verifiers can extract all necessary data directly from Bitcoin without requiring off-chain storage lookups.

---

## Anchor Types

| Anchor Type | Can Embed | Purpose |
|-------------|-----------|---------|
| **Capsule Anchor** | Full capsule metadata | Timestamp capsule existence |
| **Market Definition Anchor** | Full MarketDefinition | Make market rules immutable and self-verifiable |
| **Witness Set Anchor** | Full WitnessSet config | Track witness configuration changes |
| **Stake Registration** | Participant return addresses | Enable on-chain winner identification |

---

## Payload Strategy

### Recommended Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                    PAYLOAD STRATEGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Small objects (≤256 bytes):                                    │
│  └─> Embed FULL DATA directly in OP_RETURN                      │
│  └─> Self-contained verification, no off-chain lookup           │
│                                                                  │
│  Medium objects (256 bytes - 2KB):                              │
│  └─> Embed full data OR use commitment based on use case        │
│  └─> Consider transaction fee economics                          │
│                                                                  │
│  Large objects (>2KB):                                          │
│  └─> Use hash commitment + off-chain storage                    │
│  └─> More economical, but requires data availability            │
│                                                                  │
│  ALL objects:                                                    │
│  └─> Include hash commitment for tamper-evidence                │
│  └─> Even if full data embedded, hash enables quick validation  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Payload Format

```
OP_RETURN payload:

┌──────────┬─────────────┬──────────────────────────┬─────────────────┐
│ Version  │ Anchor Type │ Content Hash (optional)  │ Full Data       │
│ (1 byte) │ (1 byte)    │ (32 bytes if present)    │ (variable)      │
└──────────┴─────────────┴──────────────────────────┴─────────────────┘

Flags in Version byte:
  bit 0: 1 = includes content hash
  bit 1: 1 = includes full data
  bit 2-7: version number (0-63)
```

### Anchor Type Codes

```
0x01 = CAPSULE_ANCHOR
0x02 = MARKET_DEFINITION_ANCHOR
0x03 = WITNESS_SET_ANCHOR
0x04 = REDEMPTION_ANCHOR
0x05 = STAKE_REGISTRATION (NEW - for Q4 resolution)
```

---

## Self-Contained Market Anchors

With the limit removed, a complete MarketDefinition can be embedded on-chain:

```
MarketDefinition (~150-200 bytes typical):
├── version: 1 byte
├── capsuleCIDCommit: 32 bytes
├── predicateType: 1 byte
├── predicateParams: ~20-50 bytes (variable)
├── resolutionHeight: 4 bytes
├── witnessSetHash: 32 bytes
├── threshold: 1 byte
├── payoutRule: 1 byte
└── feePolicy: ~10-20 bytes

Total: ~100-150 bytes (easily fits in OP_RETURN)
```

**Benefit:** Anyone with the Bitcoin transaction can verify market rules without needing IPFS/Arweave/off-chain lookup.

---

## Anchoring Workflows

### Capsule Anchor (Self-Contained)

```
1. Build capsule object
2. Serialize capsule metadata (pubkey, enc_params, attestation_type)
   └─> Note: enc_privkey stored off-chain (too large)
3. Compute capsuleHash = SHA256(full_capsule)
4. Broadcast Bitcoin TX:
   - Output 0: OP_RETURN <version | 0x01 | capsuleHash | capsule_metadata>
5. Full capsule (with enc_privkey) stored off-chain, referenced by capsuleHash
```

### Market Definition Anchor (Fully On-Chain)

```
1. Build MarketDefinition
2. Serialize to canonical bytes (~150 bytes)
3. Compute marketHash = SHA256(MarketDefinition)
4. Broadcast Bitcoin TX:
   - Output 0: OP_RETURN <version | 0x02 | marketHash | MarketDefinition>
5. Market is FULLY VERIFIABLE from Bitcoin alone
```

### Stake Registration Anchor (Enables Q4 Resolution)

```
1. User prepares stake transaction
2. Include OP_RETURN with return address:
   - Output 0: Stake amount to escrow address
   - Output 1: OP_RETURN <version | 0x05 | return_address | side>
3. Payout builder scans for STAKE_REGISTRATION anchors
4. Winner list constructed entirely from on-chain data
```

This directly addresses [Q4: Winner Identification](../review/Q4-winner-identification.md).

---

## Verification

### For Self-Contained Anchors

```
1. Locate Bitcoin TX by txid
2. Parse OP_RETURN output
3. Extract embedded data
4. Verify: SHA256(embedded_data) == content_hash (if present)
5. Parse and validate embedded object
6. Done - no off-chain lookup needed
```

### For Hash-Only Anchors

```
1. Locate Bitcoin TX by txid
2. Parse OP_RETURN output
3. Extract commitment hash
4. Obtain full object from off-chain storage
5. Verify: SHA256(object) == commitment
6. Anchor height = TX confirmation height
```

---

## Design Rationale

### Why Full Embedding Now?

| Approach | Pros | Cons |
|----------|------|------|
| Hash commitment only | Minimal on-chain footprint | Requires off-chain availability |
| Full data embedding | Self-contained, censorship-resistant | Higher transaction fees |
| Hybrid (hash + data) | Best of both worlds | Slightly larger payload |

**This protocol uses hybrid approach:**
- Critical verification data embedded directly
- Large blobs (encrypted keys, proofs) stored off-chain
- Hash commitment always present for integrity

### Why OP_RETURN Over Alternatives?

| Method | UTXO Impact | Standardness | Data Size |
|--------|-------------|--------------|-----------|
| OP_RETURN | None (prunable) | Standard (v30+) | **Unlimited** |
| Inscription (Ordinals) | Creates UTXO | Varies | Large |
| P2SH/P2WSH embedding | Creates UTXO | Non-standard | Medium |
| Witness data | None | Standard | Large but expensive |

OP_RETURN remains the cleanest method - now without size constraints.

---

## Transaction Fee Considerations

Larger OP_RETURN payloads increase transaction size and fees:

| Payload Size | Approx TX Size | Fee @ 10 sat/vB |
|--------------|----------------|-----------------|
| 34 bytes (hash only) | ~150 vB | ~1,500 sats |
| 150 bytes (MarketDef) | ~270 vB | ~2,700 sats |
| 500 bytes (full capsule meta) | ~620 vB | ~6,200 sats |
| 2 KB | ~2,150 vB | ~21,500 sats |

**Recommendation:** Embed critical verification data; use off-chain for large blobs.

---

## Off-Chain Storage (For Large Objects)

When hash commitments are used, objects MUST be retrievable:

| Storage | Availability | Cost | Best For |
|---------|--------------|------|----------|
| IPFS + Pinning | High (if pinned) | Low | Encrypted keys, proofs |
| Arweave | Permanent | Medium | Archival, immutable data |
| Nostr | Distributed | Free | Real-time, social distribution |
| Protocol-specific DHT | Protocol-dependent | Free | Witness coordination |

The protocol does not mandate a specific storage layer, but implementations SHOULD support multiple backends for redundancy.

---

## Migration Note

Existing implementations targeting pre-v30 Bitcoin Core should:
1. Continue supporting hash-only anchors (backward compatible)
2. Add support for parsing embedded data when present
3. Optionally upgrade to embedded anchors for new markets

---

## Related Documents

- [Market Definition](03-market-definition.md) - What gets anchored
- [Witness Network](05-witness-network.md) - Witness set anchoring
- [Q4: Winner Identification](../review/Q4-winner-identification.md) - Stake registration use case
- [Appendix: Serialization](../appendix/serialization.md) - Canonical encoding
