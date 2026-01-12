# Escrow System

> Taproot escrows and payout construction

[< Back to Market Definition](03-market-definition.md) | [Next: Witness Network >](05-witness-network.md)

---

## Escrow Model

Each market has **two Taproot escrow addresses**:

- **YES escrow**: Participants betting YES send BTC here
- **NO escrow**: Participants betting NO send BTC here

```
Market
├── YES Escrow (Taproot address)
│   └── UTXOs from YES bettors
└── NO Escrow (Taproot address)
    └── UTXOs from NO bettors
```

---

## Taproot Spend Paths

Each escrow output includes multiple spend paths:

```
Taproot Output Structure
│
├── Key Path (Internal Key)
│   └── Cooperative close / emergency recovery
│
└── Script Tree (Merkle Root)
    ├── YES Branch
    │   └── Requires witness threshold sig over YES outcome
    ├── NO Branch
    │   └── Requires witness threshold sig over NO outcome
    └── Timeout Branch (optional)
        └── Refund after timeout height
```

### Key Path (Cooperative Spend)

Allows cooperative close by:
- Pre-committed market operator key, OR
- Participant aggregate key (MuSig2)

Use cases:
- Early cancellation (all parties agree)
- Refunds before resolution
- Emergency recovery

### Script Path: YES Outcome

```
<WPK> OP_CHECKSIG
```

Where:
- `WPK` = x-only aggregate witness public key
- Signature must be over `OutcomeMessage(marketHash, YES, resolutionHeight)`

### Script Path: NO Outcome

```
<WPK> OP_CHECKSIG
```

Same as YES, but signature over `OutcomeMessage(marketHash, NO, resolutionHeight)`.

### Timeout Branch (Optional)

```
<timeout_height> OP_CHECKLOCKTIMEVERIFY OP_DROP
<refund_key> OP_CHECKSIG
```

Allows refunds if witnesses fail to produce a signature.

---

## Outcome Attestation Messages

### OutcomeMessage Structure

```
OutcomeMessage {
  marketHash: bytes32
  outcome: bool          // YES=1, NO=0
  resolutionHeight: u32
}
```

### Signing

Witnesses sign:

```
msgHash = SHA256(OutcomeMessage)
```

Using threshold Schnorr signature (FROST or MuSig2).

---

## Taproot Script Construction

### Address Generation

```python
def generate_escrow_address(market, witness_aggregate_pk, side):
    # Internal key (for cooperative spend)
    internal_key = compute_internal_key(market)

    # Script tree
    yes_script = build_outcome_script(witness_aggregate_pk, YES)
    no_script = build_outcome_script(witness_aggregate_pk, NO)
    timeout_script = build_timeout_script(market.timeout_height)

    # Build Merkle tree
    tree = TapTree([yes_script, no_script, timeout_script])

    # Taproot output key
    output_key = taproot_output_key(internal_key, tree.root)

    return bech32m_encode(output_key)
```

### Script Template (High Level)

Let `WPK` be the x-only witness aggregate public key.

**YES branch** requires Schnorr signature `sigYes` verifying `WPK` over:
```
OutcomeMessage(marketHash, YES, resolutionHeight)
```

**NO branch** requires Schnorr signature `sigNo` verifying `WPK` over:
```
OutcomeMessage(marketHash, NO, resolutionHeight)
```

---

## Payout Construction

### Winner Determination

After threshold signature exists for the resolved outcome:

1. Identify winning side (YES or NO)
2. Collect all UTXOs from winning escrow
3. Collect all UTXOs from losing escrow (losers' funds go to winners)
4. Build payout transaction

### Pro-Rata Payout (MVP)

```
Total pool = YES escrow + NO escrow - fees

For each winner i:
  payout[i] = (stake[i] / total_winning_stake) * (total pool)
```

### Payout Transaction Structure

```
Inputs:
  - Winning escrow UTXO(s) via YES/NO script path
  - Losing escrow UTXO(s) via YES/NO script path

Outputs:
  - Winner 1: payout[1] sats
  - Winner 2: payout[2] sats
  - ...
  - Witness pool: witness_fee sats
  - Protocol treasury: protocol_fee sats
  - (Optional) Burn output: burn_fee sats (OP_RETURN)
```

### Who Builds the Payout Transaction?

The payout builder can be:
- Any participant (permissionless)
- A coordinator service
- An automated aggregator

**Important:** The aggregator is not trusted; invalid payout construction fails on-chain verification.

---

## No Admin Resolution

No manual dispute resolution exists. If witness network does not sign:
- Escrow cannot be spent via outcome paths
- Timeout branch (if present) allows refunds after delay

This is a feature, not a bug: **deterministic predicates should not require disputes**.

---

## Open Questions

### Q3: UTXO Stability During Resolution

> See [Q3: UTXO Stability](../review/Q3-utxo-stability.md)

If stakes are deposited close to resolution height and a reorg occurs, payout transaction may reference invalid UTXOs.

### Q4: Winner Identification

> See [Q4: Winner Identification](../review/Q4-winner-identification.md)

Taproot escrows don't track who contributed what. How do we build the winner output list?

**Options:**
1. Off-chain registry with merkle proof
2. OP_RETURN in stake transactions with return address
3. Escrow coordinator with transparency log
4. Simplify to single-winner model

---

## Implementation Checklist

- [ ] Taproot address generator for YES/NO escrows
- [ ] Script path templates (YES, NO, timeout)
- [ ] OutcomeMessage serialization
- [ ] Payout builder with fee calculation
- [ ] UTXO aggregation from escrow addresses
- [ ] Signature verification for script spend

---

## Related Documents

- [Market Definition](03-market-definition.md) - Market structure and predicates
- [Witness Network](05-witness-network.md) - Who produces the threshold signature
- [Appendix: Taproot Scripts](../appendix/taproot-scripts.md) - Full script templates
