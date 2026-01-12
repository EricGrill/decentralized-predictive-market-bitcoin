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

### Option F: Automated Market Maker (AMM/Bonding Curve)

Replace discrete betting with continuous price discovery:

```
Mechanism:
- Price set by bonding curve, not order book
- Each trade moves price along curve
- Large trades = high slippage
- Capsule holder's trades naturally move price against them

Bonding curve example:
  price = k * (yes_reserve / no_reserve)

If holder tries to manipulate:
1. Holder buys large NO position
2. Price moves significantly (slippage)
3. Holder's average cost is much higher
4. Even if they win, profit margin compressed

Self-correcting property:
- Market prices in holder's power automatically
- If holder controls outcome, market will price that in
- Sophisticated traders arbitrage manipulation attempts
```

**Pros:**
- Always liquid (no counterparty matching)
- Self-correcting prices
- Large manipulation = large slippage cost
- Well-understood DeFi primitive

**Cons:**
- Impermanent loss for liquidity providers
- Different UX than traditional betting
- May not fully prevent small-scale manipulation
- Complexity for users

### Option G: Blind Betting (Commit-Reveal)

Hide bet direction until betting period closes:

```
Two-phase betting:

COMMIT PHASE (blocks 0 - H):
1. User creates: commitment = hash(direction || amount || salt)
2. User broadcasts commitment on-chain
3. No one knows bet direction, only that bets exist

REVEAL PHASE (blocks H - H+R):
1. Users reveal: (direction, amount, salt)
2. Protocol verifies: hash matches commitment
3. Revealed bets are counted
4. Unrevealed commitments are forfeited

Resolution:
- After reveal phase, odds are known
- Capsule holder saw NO information during commit phase
- Cannot make informed manipulation decision
```

**Pros:**
- Zero information leakage during betting
- Holder cannot react to market sentiment
- Simple cryptographic construction
- Battle-tested in other protocols

**Cons:**
- Two-transaction overhead
- Must return for reveal (or lose stake)
- Griefing: commit but don't reveal
- Slight UX friction

### Option H: Time-Weighted Stakes

Earlier stakes count more, reducing last-minute manipulation:

```
Weight formula:
  weight(stake) = (resolution_height - stake_height) / window_size

Example (100-block window):
- Stake at block -100: weight = 1.0 (full weight)
- Stake at block -50:  weight = 0.5 (half weight)
- Stake at block -10:  weight = 0.1 (10% weight)
- Stake at block -1:   weight = 0.01 (negligible)

Payout calculation:
  payout[i] = stake[i] * weight[i] / sum(stake * weight)

Implication for manipulation:
- Holder sees 80% YES at block -50
- Holder bets large on NO at block -10
- Holder's NO stake has low weight (0.1)
- Even winning NO, holder gets small payout
```

**Pros:**
- Rewards early conviction
- Punishes late (informed) manipulation
- Gradual, no hard cutoff
- Simple to implement

**Cons:**
- Discourages late participation
- May reduce total liquidity
- Adds complexity to payout math
- Holder could still bet early if they planned ahead

### Option I: Prediction Market Insurance

Third-party insurance against manipulation:

```
Insurance market:

1. Insurer offers "manipulation protection" policy
2. Bettors pay small premium (e.g., 1% of stake)
3. If outcome appears manipulated, claim triggered
4. Insurer pays out (partial or full recovery)

Manipulation detection criteria:
- Outcome contradicts overwhelming pre-market evidence
- Capsule holder placed large opposing bet
- Suspicious timing patterns

Insurer incentives:
- Profitable if manipulation is rare
- Incentivized to investigate and expose manipulators
- Creates market for manipulation detection
```

**Pros:**
- Market-based solution
- Creates professional manipulation detection
- Participants can hedge manipulation risk
- Doesn't restrict market types

**Cons:**
- Defining "manipulation" is subjective
- Moral hazard (insurer might not pay)
- Insurance pricing is complex
- Requires insurance infrastructure

### Option J: Capsule Controller Staking Requirement

Mandatory collateral from controller:

```
Market creation requirement:
1. Controller must stake C = f(market_cap) as collateral
2. Collateral locked until resolution + challenge period
3. If manipulation proven, collateral slashed
4. If no challenge, collateral returned

Collateral formula:
  C = max(min_collateral, market_cap * 0.2)

Example:
- Market cap: 10 BTC
- Required collateral: 2 BTC
- If controller manipulates and wins 1 BTC
- But loses 2 BTC collateral
- Net loss: 1 BTC (manipulation unprofitable)

Challenge mechanism:
1. Anyone can challenge with evidence
2. Evidence: controller bet large + controlled outcome
3. Arbitration by witness set or community vote
4. Successful challenge = collateral to challenger + victims
```

**Pros:**
- Direct financial penalty for manipulation
- Makes manipulation unprofitable by design
- Controller has literal skin in the game
- Clear enforcement mechanism

**Cons:**
- Requires capital from controller
- Challenge/arbitration complexity
- What constitutes "proof" of manipulation?
- May deter legitimate capsule creation

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
