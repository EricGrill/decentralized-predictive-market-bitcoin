# Q4: Winner Identification

> How do we know who staked what when building payouts?

[< Back to Review Index](README.md)

---

## Summary

The spec calls for pro-rata payouts among winners, but Taproot escrow UTXOs don't inherently track contributor identity or amounts. Without a solution, payouts cannot be constructed.

---

## Severity

**Critical**

## MVP Blocker

**Yes** - Cannot ship without solving this.

---

## The Question

From the spec (Section 11.2):
> "For MVP, payouts are pro-rata among winning side's escrow contributors."

But Taproot escrows are just Bitcoin addresses. When someone sends BTC to the YES escrow:
- The escrow gains a UTXO
- The UTXO has an amount
- **The UTXO does NOT record who sent it or where to pay winnings**

**How does the payout builder know:**
1. Which addresses contributed to each escrow?
2. How much each contributor staked?
3. Where to send each winner's payout?

---

## Why This Matters

This is a **fundamental architectural question**, not an edge case.

Without winner identification:
- ❌ Cannot construct payout outputs
- ❌ Cannot enforce pro-rata distribution
- ❌ Cannot build the payout transaction at all

The market literally cannot resolve.

---

## Technical Analysis

### What On-Chain Data Tells Us

For each escrow UTXO, we can determine:
- ✅ Amount (sats)
- ✅ ScriptPubKey (the escrow address)
- ✅ Funding transaction (txid)
- ⚠️ Funding transaction inputs (but these could be from exchange, mixer, etc.)

We CANNOT determine:
- ❌ Intended recipient for winnings
- ❌ Participant identity
- ❌ Return address preference

### The Coin Selection Problem

Even if we trace funding tx inputs:

```
Stake transaction:
  Input 0: from address A (but this could be exchange hot wallet)
  Input 1: from address B (change from previous tx)
  Output 0: 0.5 BTC to YES escrow
  Output 1: change to address C

Where should winnings go? A? B? C? Somewhere else entirely?
```

Bitcoin's UTXO model doesn't preserve this information.

---

## Proposed Resolutions

### Option A: Off-Chain Registry with Merkle Commitment

Maintain an off-chain registry, commit its root in market definition:

```
StakeRegistry {
  entries: [
    { pubkey: "abc...", amount: 50000000, side: YES, return_addr: "bc1..." },
    { pubkey: "def...", amount: 30000000, side: NO, return_addr: "bc1..." },
    ...
  ]
}

registry_root = MerkleRoot(entries)
```

**Market definition includes:** `stake_registry_commitment: registry_root`

**Stake flow:**
1. User registers stake intent with registry (off-chain)
2. User sends BTC to escrow
3. Registry operator verifies and records
4. At payout: merkle proof shows inclusion

**Pros:**
- Clean separation of concerns
- Scales to many participants

**Cons:**
- Who runs the registry?
- Trust registry operator
- Registry availability = market liveness

### Option B: OP_RETURN in Stake Transaction

Require stake transactions to include return address in OP_RETURN:

```
Stake transaction:
  Output 0: 0.5 BTC to YES escrow
  Output 1: OP_RETURN <return_address>
  Output 2: change (optional)
```

**Payout builder scans:**
1. All txs sending to escrow address
2. Parse OP_RETURN for return address
3. Build (return_addr, amount) list

**Pros:**
- Fully on-chain
- No off-chain infrastructure
- Permissionless verification

**Cons:**
- Larger stake transactions
- OP_RETURN size limits (80 bytes)
- Must enforce format

### Option C: Escrow Coordinator with Transparency Log

Designated coordinator manages escrow participation:

```
Stake flow:
1. User requests stake with coordinator
2. Coordinator provides deposit address (unique per user)
3. User sends BTC
4. Coordinator consolidates to escrow (optional)
5. Coordinator maintains public log of all stakes

Transparency:
- Coordinator publishes merkle tree of stakes
- Anyone can verify their inclusion
- Coordinator cannot lie about totals (escrow balance is public)
```

**Pros:**
- Familiar UX (deposit addresses)
- Coordinator handles complexity
- Transparency log provides accountability

**Cons:**
- Coordinator is trusted party
- Single point of failure
- Regulatory concerns (custody)

### Option D: Simplified Model (Single Winner / Lottery)

Avoid the problem entirely:

```
Instead of pro-rata:
- Lottery: random winner weighted by stake
- Or: single largest stake wins all (chicken game)
```

**Pros:**
- Dramatically simpler
- Single output in payout tx
- Winner self-identifies by signing

**Cons:**
- Changes market economics fundamentally
- Discourages small participants
- Not what users expect from "prediction market"

### Option E: Stake NFTs / Receipts

Issue receipt tokens for each stake:

```
Stake flow:
1. User sends BTC to escrow
2. Escrow operator issues signed receipt:
   Receipt { stake_txid, amount, side, return_addr, signature }
3. User holds receipt
4. At payout: user submits receipt to claim winnings

Verification:
- Receipt signature valid
- stake_txid exists and sends to escrow
- Not already claimed
```

**Pros:**
- User controls their claim
- No centralized payout construction

**Cons:**
- User must safeguard receipt
- Lost receipt = lost funds
- Claim coordination complexity

---

## Recommended Approach

**MVP:** Option C - Coordinator with transparency log

Rationale:
- Simplest to implement
- Acceptable trust assumption for MVP
- Transparency log provides accountability
- Can migrate to more decentralized options later

```
MVP Implementation:

1. Market creator designates coordinator pubkey
2. Coordinator operates:
   - Stake registration endpoint
   - Deposit address generation
   - Public stake log (append-only)
   - Payout construction service

3. Transparency guarantees:
   - Stake log is merkle tree, root published to Nostr/IPFS
   - Anyone can verify their stake inclusion
   - Escrow balance = sum of logged stakes (auditable)

4. Trust model:
   - Coordinator cannot steal (doesn't control escrow keys)
   - Coordinator CAN censor (refuse to log stakes)
   - Coordinator CAN misattribute (wrong return address)
   - Mitigation: reputation, legal, competition
```

**Future:** Migrate to Option B (OP_RETURN) or Option A (decentralized registry)

```
Decentralization path:
1. MVP: Trusted coordinator
2. v2: Multiple competing coordinators
3. v3: Pure on-chain with OP_RETURN
4. v4: Decentralized registry with economic incentives
```

---

## Acceptance Criteria

This question is resolved when:

- [ ] Stake registration mechanism specified
- [ ] Payout builder can deterministically construct winner list
- [ ] Trust assumptions documented
- [ ] Coordinator API defined (for MVP)
- [ ] Migration path to decentralized solution outlined
- [ ] Test: complete market lifecycle with payouts working

---

## Related

- [Spec: Escrow System](../spec/04-escrow-system.md#payout-construction)
- [Q3: UTXO Stability](Q3-utxo-stability.md) - Must know when stakes finalize
- [Q2: Witness Economics](Q2-witness-economics.md) - Fee distribution also needs winner list
