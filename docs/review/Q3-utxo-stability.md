# Q3: UTXO Stability During Resolution

> What happens if escrow deposits reorg after witnesses sign?

[< Back to Review Index](README.md)

---

## Summary

Witnesses wait K confirmations before evaluating predicates, but there's no equivalent rule for escrow deposits. A reorg affecting stake transactions could invalidate the payout transaction.

---

## Severity

**High**

## MVP Blocker

**No** - K-confirmation rule for stakes is a straightforward mitigation.

---

## The Question

From the spec (Section 10.5):
> "Witnesses MUST wait K confirmations before signing"

This protects predicate evaluation from reorgs. But the same concern applies to escrow deposits:

1. Alice stakes 1 BTC to YES escrow in block N
2. Payout builder notes Alice's stake
3. Reorg removes block N (and Alice's stake)
4. Payout transaction references non-existent UTXO
5. Payout fails on broadcast

**How does the protocol handle escrow UTXO stability?**

---

## Why This Matters

Payout transactions reference specific UTXOs by outpoint (`txid:vout`). If those UTXOs change:

- **Reorgs**: Stake transaction orphaned, UTXO doesn't exist
- **Double-spend attempts**: Attacker tries to reclaim stake before resolution
- **RBF replacement**: Stake tx replaced with different outputs

Any of these breaks payout construction.

### Edge Case: Late Stakes

Most problematic when stakes arrive close to resolution height:

```
Timeline:
Block H-10: Market created
Block H-5:  Alice stakes (shallow confirmation)
Block H:    Resolution height
Block H+K:  Witnesses sign
Block H+K+1: Reorg removes H-5 through H
Block H+K+2: Payout tx fails (Alice's stake gone)
```

---

## Technical Analysis

### Current State

The spec doesn't address stake finality requirements. Implicit assumptions:
- Stakes "just exist" in escrow addresses
- Payout builder scans escrow UTXOs at resolution time
- No mention of confirmation requirements

### Attack Vectors

**1. Intentional Double-Spend**
```
1. Stake to YES escrow (low fee, likely to be dropped)
2. Wait for market to resolve YES
3. If YES wins: let stake confirm (win payout)
4. If NO wins: double-spend stake back to self (avoid loss)
```

This is classic "nothing at stake" for prediction markets.

**2. Accidental Reorg**
```
1. Many stakes arrive in same block
2. That block gets reorged
3. Massive payout failure
```

**3. MEV-style Manipulation**
```
1. See pending resolution signature
2. Front-run with stake removal
3. (Less relevant for Bitcoin, but possible with RBF)
```

---

## Proposed Resolutions

### Option A: Stake Finality Window

Require stakes to confirm K blocks before resolution height:

```
Stake acceptance rule:
  stake_valid = (stake_confirm_height + K_stake) < resolution_height

Where:
  K_stake = required confirmations for stake finality

Recommendation:
  K_stake = K_predicate (same as witness confirmation rule)
```

**Implementation:**
- Payout builder ignores stakes confirmed after `resolution_height - K_stake`
- Late stakes are refunded (not included in market)
- Clear cutoff communicated to participants

**Pros:**
- Simple, deterministic rule
- Aligns with existing K-confirmation pattern

**Cons:**
- Late participants excluded
- Requires stake tracking with confirmation heights

### Option B: Dynamic Payout Construction

Build payout transaction AFTER resolution, using only stable UTXOs:

```
Payout construction flow:
1. Witnesses sign outcome (at H + K)
2. Wait additional M blocks for stake stability
3. Scan escrow UTXOs at H + K + M
4. Build payout referencing only confirmed UTXOs
5. Broadcast payout
```

**Pros:**
- More inclusive (accepts later stakes)
- Adapts to actual UTXO state

**Cons:**
- Adds delay M to payout
- More complex coordination

### Option C: Escrow Aggregation

Single coordinator aggregates stakes into one UTXO per side:

```
Stake flow:
1. User sends BTC to coordinator
2. Coordinator adds to aggregate UTXO
3. Aggregate UTXO is the escrow

Payout flow:
1. Single UTXO per side (stable, deep confirmations)
2. Payout from aggregate
3. Coordinator distributes to winners
```

**Pros:**
- Simpler UTXO management
- Deep confirmation for aggregate

**Cons:**
- Introduces coordinator trust
- Coordinator becomes bottleneck
- Custody concerns

### Option D: Commit-Reveal Stakes

Stakes go through commit-reveal to lock in before resolution:

```
1. COMMIT: User broadcasts commitment to stake amount/side
2. REVEAL: User reveals stake (must confirm before deadline)
3. Stakes only valid if both commit and reveal confirmed

Deadline: revelation_height < resolution_height - K
```

**Pros:**
- Prevents last-minute manipulation
- Clear finality

**Cons:**
- Two-transaction overhead
- Worse UX

---

## Recommended Approach

**MVP:** Option A - Simple stake finality window

```
Parameters:
  K_stake = 6 (same as standard confirmation rule)

Rules:
  - Stakes must confirm by (resolution_height - K_stake)
  - Later stakes are not counted
  - Market documentation includes cutoff time

Implementation:
  - Payout builder filters by confirmation height
  - Returns list of (pubkey, amount, confirm_height)
  - Rejects stakes with confirm_height > cutoff
```

**Future Enhancement:** Option B for better UX

```
After MVP stability:
  - Implement dynamic payout construction
  - Accept stakes closer to resolution
  - Additional M-block stabilization period
```

---

## Acceptance Criteria

This question is resolved when:

- [ ] Stake finality rule defined (K_stake parameter)
- [ ] Payout builder implements confirmation filtering
- [ ] Late stake handling documented (refund? excluded?)
- [ ] Test vectors for edge cases (reorg, late stake, double-spend attempt)

---

## Related

- [Spec: Escrow System](../spec/04-escrow-system.md)
- [Spec: Witness Network - Reorg Handling](../spec/05-witness-network.md#reorg-handling)
- [Q4: Winner Identification](Q4-winner-identification.md) - Payout construction depends on this
