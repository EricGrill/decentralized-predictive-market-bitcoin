# Q4: Winner Identification

> How do we know who staked what when building payouts?

[< Back to Review Index](README.md)

---

## Summary

The spec calls for pro-rata payouts among winners, but Taproot escrow UTXOs don't inherently track contributor identity or amounts. Without a solution, payouts cannot be constructed.

---

## Severity

**Critical** (Resolved)

## MVP Blocker

**No** - Resolved via OP_RETURN stake registration (Bitcoin Core v30+).

## Status

**In Progress** - Solution identified, implementation pending.

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

### Option B: OP_RETURN in Stake Transaction (RECOMMENDED)

**As of Bitcoin Core v30, the OP_RETURN limit has been increased to 100KB, making this the preferred approach.**

Require stake transactions to include return address in OP_RETURN:

```
Stake transaction:
  Output 0: 0.5 BTC to YES escrow
  Output 1: OP_RETURN <version | 0x05 | market_hash | return_address | side>
  Output 2: change (optional)

Payload breakdown (~70 bytes):
  version: 1 byte
  anchor_type: 1 byte (0x05 = STAKE_REGISTRATION)
  market_hash: 32 bytes (identifies which market)
  return_address: 34 bytes (P2WPKH/P2TR address)
  side: 1 byte (0x01 = YES, 0x00 = NO)
```

**Payout builder workflow:**
1. Scan all txs sending to escrow addresses
2. Parse OP_RETURN for STAKE_REGISTRATION anchors
3. Filter by market_hash
4. Build (return_addr, amount, side) list from on-chain data
5. Construct payout transaction

**Pros:**
- **Fully on-chain** - no off-chain infrastructure needed
- **Permissionless verification** - anyone can reconstruct winner list
- **Censorship resistant** - data embedded in Bitcoin
- **No coordinator trust** - eliminates single point of failure
- **Simple implementation** - scan + parse

**Cons:**
- Slightly larger stake transactions (~70 bytes extra)
- Higher transaction fees (~700 extra sats @ 10 sat/vB)
- Must enforce format (invalid OP_RETURN = stake not counted)

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

**MVP and Production: Option B - OP_RETURN Stake Registration**

With Bitcoin Core v30 removing the OP_RETURN limit (now 100KB), the on-chain approach is now the clear winner.

Rationale:
- **Fully on-chain** - no off-chain infrastructure to maintain
- **Trustless** - no coordinator, no registry operator
- **Censorship resistant** - stake registration is in Bitcoin
- **Simple** - scan escrow transactions, parse OP_RETURN, done
- **Permissionless** - anyone can verify and construct payouts

```
Implementation:

1. Define STAKE_REGISTRATION format (0x05 anchor type)
   └─> version | 0x05 | market_hash | return_address | side
   └─> ~70 bytes total

2. Stake transaction requirements:
   └─> Output 0: BTC to escrow address
   └─> Output 1: OP_RETURN with stake registration

3. Payout builder:
   └─> Scan all txs to escrow addresses
   └─> Parse OP_RETURN for valid registrations
   └─> Filter by market_hash
   └─> Sum amounts per return_address per side
   └─> Construct winner list

4. Edge cases:
   └─> Missing OP_RETURN: stake counts but unpayable (burned)
   └─> Invalid format: stake counts but unpayable (burned)
   └─> Duplicate registrations: sum amounts
```

**Fallback:** Option C (Coordinator) only if OP_RETURN scanning proves too expensive

```
Only use coordinator if:
- Bitcoin indexing is prohibitively expensive
- Real-time stake updates needed (OP_RETURN requires confirmation)
- User experience requires instant feedback
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
