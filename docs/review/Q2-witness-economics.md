# Q2: Witness Economic Model

> What incentivizes rational actors to become witnesses?

[< Back to Review Index](README.md)

---

## Summary

Witnesses must run infrastructure, post bonds, and risk slashing—but the spec doesn't define rewards. Without positive expected value, the witness network won't attract participants.

---

## Severity

**Critical**

## MVP Blocker

**No** - Can bootstrap with altruistic/aligned witnesses, but production requires economic model.

---

## The Question

From the spec:
- Witnesses MUST run Bitcoin Core full node
- Witnesses MUST maintain uptime for threshold signing
- Witnesses MUST post BTC bonds
- Witnesses face slashing for equivocation

Section 6.1 mentions `feePolicy` but doesn't specify:
- How fees are calculated
- How fees are distributed to witnesses
- What the witness share is

**Is the economic model viable for rational actors?**

---

## Why This Matters

Without clear economic incentives:

1. **Rational actors won't participate** - No ROI on infrastructure and bond
2. **Witness sets centralize** - Only entities with ulterior motives (protocol founders, large bettors) will run witnesses
3. **Market liveness fails** - If witnesses don't sign, markets don't resolve

### The Bootstrap Problem

Early markets will have low volume. Low volume means low fees. Low fees means:
- Can't attract professional witnesses
- Reliance on altruistic operators
- Fragile liveness guarantees

---

## Technical Analysis

### Cost Structure

```
Monthly costs per witness:

INFRASTRUCTURE
├── Bitcoin Core node:     $50-200/month (VPS, bandwidth, 500GB+ storage)
├── Witness daemon:        $20-100/month (additional compute, monitoring)
└── Development:           $500-2000/month (amortized dev/maintenance)

CAPITAL
├── Bond lockup:           1 BTC × 5% APY opportunity cost = $250/month
└── Slashing risk:         1% annual × 1 BTC ≈ $42/month

TOTAL:                     ~$900-2600/month per witness
```

### Revenue Requirements

For a 10-witness network to be sustainable:

```
Total monthly cost: 10 × $2000 = $20,000/month
At 70% witness allocation: Need $28,500/month total fees
At 1% fee rate: Need $2.85M/month escrow volume
At 10 markets/month: Need $285,000 average market size
```

### Volume Scenarios

| Scenario | Markets/Month | Avg Size | Total Volume | 1% Fees | Witness Share (70%) | Per Witness (N=10) |
|----------|---------------|----------|--------------|---------|---------------------|-------------------|
| Low | 10 | $10,000 | $100,000 | $1,000 | $700 | $70 |
| Medium | 100 | $50,000 | $5,000,000 | $50,000 | $35,000 | $3,500 |
| High | 500 | $100,000 | $50,000,000 | $500,000 | $350,000 | $35,000 |

**Conclusion:** Low volume scenario is not viable. Need medium+ volume or alternative incentives.

---

## Proposed Resolutions

### Option A: Pure Fee Model

```rust
struct FeePolicy {
    fee_type: FeeType,           // Fixed, proportional, or hybrid
    witness_allocation_bps: u16, // e.g., 7000 = 70%
    protocol_allocation_bps: u16,
    burn_allocation_bps: u16,
}
```

**Distribution:**
```
Total fee = fee_rate × total_escrow
Witness pool = total_fee × witness_allocation / 10000
Per witness = witness_pool / N
```

**Pros:**
- Simple, transparent
- Scales with market size

**Cons:**
- Insufficient for low-volume bootstrap phase
- High fees make small markets uneconomical

### Option B: Protocol Treasury Subsidies

Protocol maintains treasury to subsidize witness rewards during bootstrap:

```
if market_fee < min_witness_payment:
    subsidy = min_witness_payment - market_fee
    pay witnesses from treasury
```

**Pros:**
- Enables small market economics
- Controlled bootstrap period

**Cons:**
- Requires treasury funding mechanism
- Subsidy must eventually phase out

### Option C: Witness Staking Tiers

Different witness sets for different market sizes:

```
Tier 1 (< 0.1 BTC markets):
  - 3-of-5 witnesses
  - Lower bond requirement
  - Lower fees acceptable

Tier 2 (0.1-1 BTC markets):
  - 5-of-7 witnesses
  - Standard bond
  - Standard fees

Tier 3 (> 1 BTC markets):
  - 7-of-10+ witnesses
  - High bond
  - Premium fees
```

**Pros:**
- Matches security to value at risk
- Lower barrier for small markets

**Cons:**
- Complexity
- Fragmented witness pools

### Option D: Hybrid with Reputation Rewards

Witnesses earn both fees and reputation score:

```
reputation_score = f(uptime, markets_resolved, age, bond_size)

Benefits:
- Priority for high-value markets
- Higher fee share
- Protocol governance rights
```

**Pros:**
- Long-term alignment
- Non-financial incentives matter

**Cons:**
- Reputation system complexity
- Gaming resistance

---

## Recommended Approach

**Phase 1 (MVP):** Options A + B

```
Fee structure:
- 1% proportional fee on total escrow
- 70% to witness pool, 20% protocol, 10% burn
- Treasury subsidy for markets < 0.01 BTC fee

Minimum viable:
- 5 aligned witnesses (founders, partners)
- Low bond requirement (0.1 BTC)
- Accept reduced decentralization
```

**Phase 2 (Growth):** Add Option C

```
Introduce tiers:
- Tier 1: 3-of-5, 0.01 BTC bond, small markets
- Tier 2: 5-of-7, 0.1 BTC bond, standard markets
- Tier 3: 7-of-10, 1 BTC bond, large markets

Phase out treasury subsidies as volume grows.
```

**Phase 3 (Mature):** Add Option D

```
Reputation system:
- Track witness performance
- Premium markets require high reputation
- Long-term alignment incentives
```

---

## Acceptance Criteria

This question is resolved when:

- [ ] FeePolicy encoding is fully specified
- [ ] Fee distribution algorithm documented
- [ ] Bootstrap incentive mechanism defined
- [ ] Break-even analysis shows viable path
- [ ] At least 5 committed witness operators identified for MVP

---

## Related

- [Spec: Market Definition](../spec/03-market-definition.md#fee-policy)
- [Spec: Witness Network](../spec/05-witness-network.md#economic-incentives)
- [Q4: Winner Identification](Q4-winner-identification.md) - Affects payout construction
