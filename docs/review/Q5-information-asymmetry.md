# Q5: Information Asymmetry

> Can capsule holders manipulate outcomes they're betting on?

[< Back to Review Index](README.md)

---

## Summary

Some predicates depend on actions the capsule holder controls (e.g., "will capsule be redeemed?"). This creates information asymmetry where the holder can observe market odds and then decide the outcome.

---

## Severity

**High**

## MVP Blocker

**No** - Can restrict to "uncontrollable" predicates for MVP.

---

## The Question

From the end-to-end example (Section 16):
> Market: "Will capsule X be redeemed before block height H?"
> Predicate: `OP_RETURN_EXISTS(R, beforeHeight=H)`

The redemption anchor `R` must be broadcast by someone. If the capsule holder controls this:

1. Holder sees market: 80% YES (redemption expected)
2. Holder realizes they control redemption
3. Holder bets heavily on NO
4. Holder refuses to redeem before H
5. Market resolves NO
6. Holder collects winnings

**Is there an intended separation between "capsule controller" and "market participant"?**

---

## Why This Matters

This is a classic oracle manipulation pattern:

> The entity with private information about the outcome can always profit by betting against the crowd, then causing the unexpected outcome.

### Affected Predicates

| Predicate | Controllable By |
|-----------|-----------------|
| `OP_RETURN_EXISTS(commitment, H)` | Whoever can broadcast the commitment |
| `TX_CONFIRMED(txid, H)` | Whoever controls the signing keys |
| `SUCCESSOR_CAPSULE_ANCHORED(...)` | Capsule controller |
| `ADDRESS_BALANCE_GE(addr, sats, H)` | Whoever controls the address |
| `UTXO_EXISTS(outpoint, H)` | Whoever can spend or not spend |

### Uncontrollable Predicates

| Predicate | Why Uncontrollable |
|-----------|-------------------|
| `BLOCK_HEIGHT_REACHED(H)` | No one controls Bitcoin block production* |
| `TX_IN_BLOCK(txid, specific_block)` | Can't predict which block |
| External oracle attestation | Depends on oracle independence |

*Miners have some influence but not deterministic control.

---

## Technical Analysis

### Information Advantage

The capsule holder has two advantages:

**1. Knowledge of Intent**
```
Holder knows: "I will definitely redeem" or "I will not redeem"
Market knows: Probabilistic estimate based on public info
```

**2. Ability to Change Intent**
```
Before market opens: Holder intended to redeem
After seeing odds: Holder changes mind based on profit opportunity
```

### Game Theory

If holder can profit more from manipulation than from legitimate capsule use:

```
Legitimate value: V_capsule (value of what's in the capsule)
Manipulation profit: M = stake × (1 - market_odds)

If M > V_capsule: Rational to manipulate
```

For high-value markets or low-value capsules, manipulation is rational.

### Timing Attack

Even without malicious intent:

```
1. Holder plans to redeem at height H-100
2. Holder sees market odds: 95% YES
3. Holder thinks: "Low upside betting YES, but I could bet NO and delay..."
4. Temptation to delay and profit
```

The existence of the market changes holder incentives.

---

## Proposed Resolutions

### Option A: Restrict Predicates (MVP)

Only allow predicates that NO participant controls:

```
Allowed (MVP):
- BLOCK_HEIGHT_REACHED(H) — trivial but safe
- DIFFICULTY_GE(target, atHeight) — no one controls
- HASHRATE_INDICATOR(metric, threshold) — observable but uncontrollable

Prohibited (MVP):
- OP_RETURN_EXISTS where capsule holder can broadcast
- TX_CONFIRMED where capsule holder controls keys
- Any predicate where a market participant has outsized influence
```

**Pros:**
- Eliminates manipulation vector
- Simple rule to enforce

**Cons:**
- Severely limits market types
- Most interesting predicates are prohibited

### Option B: Commitment Before Market Opens

Require capsule holder to commit to action before betting begins:

```
Timeline:
1. Holder publishes: "I commit to redeem before H" (signed)
2. Commitment anchored on Bitcoin
3. Market opens for betting
4. If holder doesn't follow commitment: reputation damage, possible slashing

Enforcement:
- Commitment is public, creates accountability
- Holder can still violate but faces consequences
- Market participants know holder's stated intent
```

**Pros:**
- Preserves interesting predicates
- Creates accountability

**Cons:**
- Holder can still renege (commitment isn't enforceable)
- Only works with identified holders
- Requires off-chain reputation system

### Option C: Holder Exclusion

Capsule holder is prohibited from participating in markets on their capsule:

```
Rules:
- Capsule controller identity must be disclosed
- Controller addresses flagged
- Stakes from controller addresses rejected
- Controller cannot be witness for their markets

Enforcement:
- Social: disclosed identity faces reputation risk
- Technical: address-based filtering (imperfect, can use fresh addresses)
```

**Pros:**
- Direct approach to conflict of interest

**Cons:**
- Easy to circumvent (use different address)
- Reduces liquidity
- Privacy concerns with identity disclosure

### Option D: Skin-in-the-Game Requirement

Require capsule holder to stake on one side:

```
Market creation rules:
- Holder must stake X% of market cap on declared outcome
- If holder stakes YES: they're betting redemption happens
- Their stake is larger than manipulation profit opportunity

Example:
- Holder stakes 10 BTC on YES (redemption)
- Market cap is 20 BTC total
- Holder's position makes manipulation unprofitable
```

**Pros:**
- Aligns incentives
- Holder has skin in the game

**Cons:**
- Requires capital from holder
- May not fully eliminate manipulation for large markets

### Option E: Time-Locked Predicates

Predicate outcome must be determined before market opens:

```
Example:
- At height H0: Market asks "Did event X happen before H0?"
- Market opens at H0+100
- Outcome is already determined, just not known
- No one can change the past

Limitation:
- Only works for historical questions
- Can't bet on future actions
```

**Pros:**
- Completely eliminates manipulation

**Cons:**
- Only historical predicates
- Less interesting market types

---

## Recommended Approach

**MVP:** Option A - Restrict to uncontrollable predicates

```
MVP predicate whitelist:
1. BLOCK_HEIGHT_REACHED(H) — basic, safe
2. External facts with trusted oracle — e.g., sports, elections
3. Cross-capsule predicates — where controller of X can't control Y

Rationale:
- Eliminates manipulation entirely
- Proves protocol works for safe cases
- Expands predicate set in later phases
```

**Phase 2:** Option B + C - Commitments with identity disclosure

```
For controllable predicates:
- Holder must disclose identity
- Holder must commit to intended action
- Holder excluded from betting (best effort)
- Community can assess holder reputation
```

**Phase 3:** Option D - Skin-in-the-game markets

```
Premium market type:
- Holder stakes collateral on outcome
- Higher trust, higher liquidity allowed
- Manipulation unprofitable if stake > potential gain
```

---

## Acceptance Criteria

This question is resolved when:

- [ ] Predicate classification (controllable vs uncontrollable) documented
- [ ] MVP whitelist defined
- [ ] Rules for controllable predicates in future phases outlined
- [ ] Market creation tooling enforces predicate restrictions
- [ ] Documentation warns participants about information asymmetry risks

---

## Related

- [Spec: Market Definition - Predicates](../spec/03-market-definition.md#predicate-system)
- [Spec: Security Analysis](../spec/06-security-analysis.md)
- [Q1: ZK Genesis](Q1-zk-genesis.md) - Capsule holder identity relates to ZK trust
