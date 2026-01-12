# Security Analysis

> Failure modes, guarantees, and trust assumptions

[< Back to Witness Network](05-witness-network.md) | [Back to Index >](../README.md)

---

## Security Properties

### Guaranteed (under standard assumptions)

| Property | Guarantee | Assumption |
|----------|-----------|------------|
| BTC-only settlement | Staking and payouts in BTC | Bitcoin network operational |
| Deterministic definitions | Markets anchored on Bitcoin | Bitcoin consensus |
| Public verification | Anyone can verify markets and anchors | Bitcoin data availability |
| No UTXO pollution | Commitments use OP_RETURN | Standard relay policies |
| Capsule origin integrity | ZK + constrained boundary | Proof system soundness |

### Not Guaranteed

| Property | Limitation | Mitigation |
|----------|------------|------------|
| Fully trustless resolution | Witnesses are attestation layer | Economic incentives, threshold |
| Liveness under adversarial witnesses | T witnesses must cooperate | Timeout refunds, witness selection |
| Privacy | Chain surveillance possible | Not a protocol goal |

---

## Failure Modes

### 1. Witness Liveness Failure

**Scenario:** Witnesses do not produce threshold signature by resolution height.

**Impact:**
- Escrow cannot be spent via outcome paths
- Funds locked until timeout (if configured)

**Mitigations:**
- Timeout/refund path in escrow script
- Multiple witness set options
- Witness reputation tracking

**Recovery:**
```
If timeout_height reached AND no outcome signature:
  └─> Refund path becomes spendable
  └─> Participants reclaim stakes
```

### 2. Witness Cartel Attack

**Scenario:** ≥T witnesses collude to sign false outcome.

**Impact:**
- Incorrect side wins
- Honest participants lose funds

**Mitigations:**
- Large N, high T (e.g., 7-of-10)
- Independent operators (geographic, organizational)
- Bonding with slashing
- Public audits and reputation

**Detection:**
```
Post-resolution audit:
1. Anyone can verify predicate against Bitcoin data
2. If outcome contradicts facts, cartel is exposed
3. Reputation destruction (even without on-chain slashing)
```

### 3. Predicate Ambiguity

**Scenario:** Predicate has edge cases not covered by specification.

**Impact:**
- Honest witnesses disagree on outcome
- No threshold reached, or disputed outcome

**Mitigations:**
- Formal predicate specifications
- Comprehensive test vectors
- Predicate review before market creation
- Start with simple predicates (MVP)

**Example Edge Case:**
```
Predicate: ADDRESS_BALANCE_GE(addr, 1000000, height=H)

Edge case: What if address receives funds in block H itself?
- Include block H transactions? (inclusive)
- Only before block H? (exclusive)

Resolution: Specification MUST define this explicitly.
```

### 4. Reorg During Resolution

**Scenario:** Chain reorg changes predicate evaluation result after witnesses sign.

**Impact:**
- Outcome signature may contradict post-reorg state
- Payout based on orphaned chain state

**Mitigations:**
- K-confirmation rule (wait before signing)
- Re-evaluation on reorg detection
- Higher K for high-value markets

**K Selection Guide:**
```
Market value < 0.1 BTC: K=3 (30 minutes)
Market value 0.1-1 BTC: K=6 (1 hour)
Market value > 1 BTC: K=12+ (2+ hours)
```

### 5. Capsule Key Compromise (Pre-Encryption)

**Scenario:** Private key exposed before ZK proof generation.

**Impact:**
- Capsule holder can spend associated funds
- Markets betting on capsule state are manipulable

**Mitigations:**
- ZK genesis proof requirement
- Hardware attestation (Phase 2)
- MPC ceremony for high-value capsules

> See [Q1: ZK Genesis](../review/Q1-zk-genesis.md) for detailed analysis

### 6. Stake Tracking Failure

**Scenario:** Off-chain stake registry is lost, corrupted, or disputed.

**Impact:**
- Cannot construct valid payout transaction
- Winners cannot be identified

**Mitigations:**
- Multiple registry replicas
- On-chain stake commitments (OP_RETURN in stake tx)
- Escrow coordinator with transparency log

> See [Q4: Winner Identification](../review/Q4-winner-identification.md) for detailed analysis

---

## Trust Model Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                         TRUST LAYERS                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LAYER 0: Bitcoin Consensus                                         │
│  ├─ Trust: 51% honest hashrate                                     │
│  └─ Mitigation: Bitcoin's 15+ year track record                    │
│                                                                     │
│  LAYER 1: Capsule Integrity                                         │
│  ├─ Trust: ZK proof soundness + execution boundary                 │
│  └─ Mitigation: Formal verification, hardware attestation          │
│                                                                     │
│  LAYER 2: Witness Honesty                                           │
│  ├─ Trust: ≥T of N witnesses are honest                            │
│  └─ Mitigation: Economic bonding, slashing, reputation             │
│                                                                     │
│  LAYER 3: Predicate Determinism                                     │
│  ├─ Trust: Unambiguous specification                               │
│  └─ Mitigation: Formal spec, test vectors, limited predicate set   │
│                                                                     │
│  LAYER 4: Payout Construction                                       │
│  ├─ Trust: Correct winner identification                           │
│  └─ Mitigation: (UNSPECIFIED - Critical gap for MVP)               │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Attack Cost Analysis

### Witness Bribery Attack

**Goal:** Bribe T witnesses to sign false outcome.

**Cost:**
```
Minimum bribe = T × (bond_value + expected_future_revenue + reputation_value)

Example:
- T = 7 witnesses
- Bond = 1 BTC each
- Expected future revenue = 0.5 BTC (NPV)
- Reputation = ??? (hard to quantify)

Minimum bribe ≈ 7 × 1.5 BTC = 10.5 BTC

Market must have > 10.5 BTC at stake to be worth attacking.
```

### 51% Attack on Bitcoin

**Goal:** Reorg Bitcoin to change predicate outcome after resolution.

**Cost:**
```
Current Bitcoin hashrate: ~600 EH/s
Attack cost: Estimated $500M-1B+ per day

Only relevant for extremely high-value markets.
Protocol relies on Bitcoin's security model.
```

---

## Implementation Security Checklist

### Cryptographic

- [ ] ZK proof verification implemented correctly
- [ ] Schnorr signature verification (BIP340)
- [ ] FROST/MuSig2 implementation audited
- [ ] Constant-time operations where needed
- [ ] Secure random number generation

### Protocol

- [ ] Canonical serialization with test vectors
- [ ] Predicate evaluation deterministic across implementations
- [ ] Reorg handling tested
- [ ] Slashing proofs verified on-chain

### Operational

- [ ] Witness key management (HSM recommended)
- [ ] Bond UTXO monitoring
- [ ] Alerting for equivocation detection
- [ ] Backup and recovery procedures

---

## End-to-End Example

### Market: "Will capsule X be redeemed before block height H?"

**Setup:**
- Capsule CID commitment: `C`
- Redemption anchor commitment: `R`
- Resolution height: `H = 850000`

**Predicate:**
```
OP_RETURN_EXISTS(R, beforeHeight=H)
```

**Market Creation:**
1. Build MarketDefinition with `C`, predicate, `H`, witness set hash
2. Compute `marketHash = SHA256(MarketDefinition)`
3. Broadcast TX with `OP_RETURN(marketHash)`
4. Generate YES/NO escrow addresses

**Staking:**
- Alice sends 0.5 BTC to YES escrow (bets redemption happens)
- Bob sends 0.3 BTC to NO escrow (bets it doesn't)

**Resolution (at height H + 6 confirmations):**
1. Witnesses check if `R` exists in chain before H
2. If exists: sign YES outcome message
3. If not: sign NO outcome message
4. Aggregator produces threshold signature

**Payout (assuming YES wins):**
```
Total pool: 0.8 BTC
Fees (1%): 0.008 BTC
  └─ Witnesses (70%): 0.0056 BTC
  └─ Protocol (20%): 0.0016 BTC
  └─ Burn (10%): 0.0008 BTC

Alice receives: 0.792 BTC (0.8 - 0.008)
Bob receives: 0 BTC (lost bet)
```

---

## Related Documents

- [Q1: ZK Genesis](../review/Q1-zk-genesis.md) - Cryptographic origin guarantees
- [Q2: Witness Economics](../review/Q2-witness-economics.md) - Economic model
- [Q3: UTXO Stability](../review/Q3-utxo-stability.md) - Reorg handling
- [Q4: Winner Identification](../review/Q4-winner-identification.md) - Payout construction
- [Q5: Information Asymmetry](../review/Q5-information-asymmetry.md) - Predicate manipulation
