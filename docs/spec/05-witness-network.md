# Witness Network

> Decentralized attestation, bonding, slashing, and governance

[< Back to Escrow System](04-escrow-system.md) | [Next: Security Analysis >](06-security-analysis.md)

---

## Purpose

Bitcoin cannot enforce "if TX exists then pay YES" inside Script without external attestation. **Witnesses provide that attestation.**

Witnesses:
- Do NOT custody funds
- Do NOT decide meaning
- ONLY attest facts by signing the deterministic outcome message

---

## Witness Node Requirements

Each witness MUST:

| Requirement | Purpose |
|-------------|---------|
| Run Bitcoin Core full node | Access to chain state |
| Index needed data | OP_RETURN index, tx/utxo lookups |
| Evaluate predicates deterministically | At `resolutionHeight` |
| Participate in threshold signing | FROST or MuSig2 |
| Post and maintain BTC bond | Accountability |

---

## Witness Key Structure

```
Witness Set
├── N participants (individual pubkeys)
├── Threshold T (signatures required)
└── Aggregate x-only public key (on-chain verification)
```

On-chain verification uses only the **aggregate key**. Individual witness keys are for internal coordination.

---

## Threshold Signing Flow

```
1. TRIGGER
   └─> Chain tip reaches resolutionHeight

2. WAIT
   └─> Each witness waits K confirmations (reorg protection)

3. EVALUATE
   └─> Each witness independently evaluates predicate
   └─> Deterministic: all honest witnesses get same result

4. SIGN
   └─> Each witness produces partial signature for computed outcome
   └─> Uses FROST or MuSig2 protocol

5. AGGREGATE
   └─> Aggregator collects ≥T partial signatures
   └─> Produces final Schnorr signature

6. PUBLISH
   └─> Final signature made available
   └─> Anyone can build payout transaction
```

**Important:** The aggregator is NOT trusted. Invalid aggregation fails signature verification.

---

## Reorg Handling

Witnesses MUST:

- [ ] Define K confirmations in configuration
- [ ] Re-evaluate if reorg occurs before K confirmations
- [ ] Only sign once conditions are stable

### Recommended K Values

| Security Level | K Confirmations | Use Case |
|----------------|-----------------|----------|
| Low | 3 | Small markets, fast resolution |
| Medium | 6 | Standard markets |
| High | 12 | High-value markets |
| Very High | 100+ | Mission-critical |

---

## Witness Bonding

### Bond Requirement

Each witness MUST lock BTC in a **bond UTXO** that is spendable under slashing conditions.

```
Bond UTXO Structure:
├── Normal spend: witness key (for voluntary exit after delay)
└── Slash spend: slashing proof verification
```

Bond amount and script template are **protocol parameters** (versioned).

### Recommended Bond Amounts

| Market Tier | Minimum Bond | Rationale |
|-------------|--------------|-----------|
| Tier 1 (< 0.1 BTC escrow) | 0.01 BTC | Low-value markets |
| Tier 2 (0.1-1 BTC) | 0.1 BTC | Standard markets |
| Tier 3 (> 1 BTC) | 1 BTC | High-value markets |

---

## Slashing Conditions

### Slashable Offenses

| Offense | Severity | Detection |
|---------|----------|-----------|
| **Equivocation** | Critical | Two signatures for different outcomes |
| **Contradictory heights** | Critical | Same market, different resolution heights |
| **Invalid attestation** | High | Signature contradicts Bitcoin facts |

### Equivocation Proof

```
EquivocationProof {
  marketHash: bytes32
  resolutionHeight: u32
  sig_yes: Signature      // Witness signed YES
  sig_no: Signature       // Same witness signed NO
  witness_pk: PublicKey   // Identifies the equivocator
}
```

If both signatures verify under same `witness_pk` for same `marketHash`, equivocation is proven.

### Slashing Execution

A slashing transaction spends the witness bond:

```
Distribution options (protocol parameter):
├── Burn to fees (100%)
├── Pay to reporter (50%) + burn (50%)
└── Distribute to honest witnesses
```

---

## Witness Set Governance

### WitnessSet Definition

```
WitnessSet {
  version: u8
  members: [pubkey]           // List of witness public keys
  threshold: u8               // T of N
  aggregateXOnly: pubkey      // Combined public key for verification
  bondParamsCommit: bytes32   // Commitment to bond requirements
}

witnessSetHash = SHA256(WitnessSet)
```

### Witness Set Anchoring

Changes to witness sets are published via Bitcoin transaction:

```
OP_RETURN <0x01 | 0x03 | witnessSetHash>
```

### Activation Delay

A witness set update becomes active only **after D blocks**, preventing surprise takeover.

| Delay Type | D Blocks | Duration (~10 min/block) |
|------------|----------|--------------------------|
| Fast | 144 | ~1 day |
| Standard | 1008 | ~1 week |
| Secure | 4032 | ~1 month |

### Market-Scoped Witness Sets

Each market binds to a specific `witnessSetHash`. Later witness set changes **do not affect existing markets**.

```
Market A (created block 800,000)
└─> witnessSetHash: 0xabc...
    └─> Resolved by witness set 0xabc, regardless of later updates

Market B (created block 801,000)
└─> witnessSetHash: 0xdef...
    └─> Uses updated witness set 0xdef
```

---

## Economic Incentives

> See [Q2: Witness Economics](../review/Q2-witness-economics.md) for detailed analysis

### Cost Structure

```
Monthly costs per witness:
├── Infrastructure: $50-200 (VPS, bandwidth, storage)
├── Operations: $20-100 (monitoring, maintenance)
├── Development: $500-2000 (amortized)
├── Bond opportunity cost: ~$250 (1 BTC @ 5% APY)
└── Slashing risk: ~$42 (1% annual risk × 1 BTC)

Total: ~$900-2600/month
```

### Revenue Model (Proposed)

```
Per-market fee = FeePolicy.feeType applied to total escrow
Witness share = fee × witnessAllocationBPS / 10000
Per-witness = Witness share / N
```

**Break-even analysis:**
- 10 witnesses, $2000/month cost each
- Need $20,000/month in witness fees
- At 0.7% witness allocation, need ~$2.9M monthly escrow volume

---

## Cartel Risk Mitigation

Witnesses could collude to sign false outcomes. Mitigations:

| Mitigation | Effectiveness |
|------------|---------------|
| Larger N, higher T | Makes collusion harder |
| Independent operators | Geographic/organizational diversity |
| Bonding + slashing | Economic penalty for misbehavior |
| Public auditability | Reputation consequences |
| Market-level selection | Choose witness sets you trust |

---

## Implementation Checklist

- [ ] Witness daemon with Bitcoin Core RPC
- [ ] Predicate evaluation engine
- [ ] FROST/MuSig2 threshold signing
- [ ] Reorg detection and re-evaluation
- [ ] Bond UTXO management
- [ ] Equivocation detection
- [ ] Slashing transaction builder
- [ ] Witness set update workflow

---

## Related Documents

- [Escrow System](04-escrow-system.md) - What witnesses sign for
- [Security Analysis](06-security-analysis.md) - Failure modes
- [Q2: Witness Economics](../review/Q2-witness-economics.md) - Incentive analysis
- [Appendix: FROST Protocol](../appendix/frost-protocol.md) - Threshold signing details
