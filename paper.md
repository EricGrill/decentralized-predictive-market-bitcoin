# Bitcoin-Native Capsule Prediction Market Protocol (All-Inclusive)
## BTC-Denominated, BTC-Settled Markets for Verifiable Cryptographic Outcomes

**Relevant MCP Repositories (context):**
- Memvid State Service: https://github.com/EricGrill/mcp-memvid-state-service  
- Predictive Market MCP: https://github.com/EricGrill/mcp-predictive-market  
- Bitcoin CLI MCP: https://github.com/EricGrill/mcp-bitcoin-cli  

---

## 0. Abstract

This document specifies a **Bitcoin-only** predictive market protocol whose markets resolve on the future state of cryptographic “capsules” that are anchored to Bitcoin. The system is designed to be auditable without disclosure: capsules expose Bitcoin public keys for verification while keeping private keys encrypted, and markets are denominated and settled entirely in BTC.

Because Bitcoin Script cannot introspect Bitcoin’s own chain state, markets require a **decentralized witness network** (a constrained, replaceable attestation layer) that evaluates deterministic predicates over Bitcoin-observable facts and produces **threshold Schnorr signatures** to unlock Taproot escrow paths.

The protocol avoids smart-contract platforms, stablecoins, and external oracle networks. It relies on Bitcoin for time and finality; on Taproot for enforceable escrow and payout paths; on deterministic predicate definitions for unambiguous resolution; and on cryptographic origin guarantees using **zero-knowledge proofs of sealed key genesis** to prevent “tainted capsule” attacks.

---

## 1. Goals and Non-Goals

### 1.1 Goals

- **BTC-only denomination and settlement** (staking and payouts in BTC).
- **Public verifiability**: anyone can verify capsule-linked Bitcoin state (addresses, UTXOs, balances, anchors).
- **Deterministic resolution**: markets resolve from unambiguous, machine-checkable predicates.
- **No discretionary oracles**: witnesses attest facts, never interpret meaning.
- **No UTXO pollution**: commitments use OP_RETURN; escrow uses Taproot.
- **Tainted-origin prevention**: capsules MUST include zero-knowledge proofs of sealed key genesis.
- **Replaceable components**: witness sets are market-scoped and upgradeable via Bitcoin-anchored governance.

### 1.2 Non-Goals

- Retail wallet UX.
- Anonymity guarantees against blockchain analysis.
- On-chain computation or Turing-complete settlement.
- Stablecoin pegs or cross-chain bridges.
- High-frequency trading or instant resolution.

---

## 2. System Components

1. **Capsules**: structured objects that expose a Bitcoin public key and encapsulate an encrypted private key, plus proofs and policy metadata.
2. **Bitcoin Anchors**: OP_RETURN commitments that timestamp capsule existence and market definitions.
3. **Prediction Markets**: BTC-staked YES/NO markets bound to a capsule and a deterministic predicate.
4. **Taproot Escrows**: enforceable conditional payout paths for YES/NO outcomes.
5. **Decentralized Witness Network**: Bitcoin-native attestors using threshold Schnorr to authorize the correct payout path.
6. **Slashing and Governance**: Bitcoin-native bonding and penalties for witness equivocation; witness-set updates anchored on Bitcoin.

---

## 3. Capsule Model

### 3.1 Capsule Identity

A capsule is identified by a **Capsule CID** (content identifier). The CID is the capsule’s immutable identity.

### 3.2 Capsule Contents (logical)

A capsule MUST include:

- `pubkey`: Bitcoin public key in plaintext (x-only or secp256k1 pubkey as chosen).
- `enc_privkey`: encrypted Bitcoin private key.
- `enc_params`: encryption metadata (algorithm, KDF params, version).
- `zk_genesis_proof`: zero-knowledge proof of sealed key genesis (required).
- `capsule_policy`: optional governance/policy metadata (quorum rules, time locks, revocation controls).
- `provenance`: optional versioning/history references (append-only lineage).

### 3.3 Verification vs Control

- Verification: anyone can derive addresses and verify balances/UTXOs from `pubkey`.
- Control: spending is possible only if decryption policy is satisfied; private key is never revealed by the system.

---

## 4. Zero-Knowledge Key Genesis (Mandatory)

### 4.1 Problem

Encryption does not prevent compromise if a private key is exposed before encryption. A human or process can copy the key, destroying custody integrity permanently.

### 4.2 Requirement

Capsules MUST include a **zero-knowledge proof** asserting:

> “I know a private key `k` such that:
> - `pubkey = KeyDerive(k)`
> - `enc_privkey = Encrypt(k, enc_params)`
> - `k` never left a constrained execution boundary
> - `k` was never exported, logged, or observed”

The proof reveals nothing about `k` or encryption randomness.

### 4.3 Constrained Execution Boundary

Key generation and encryption MUST occur inside an environment that prevents plaintext exfiltration. Acceptable examples include:

- HSM
- secure enclave
- strict WASM sandbox with no I/O
- deterministic MPC/threshold ceremony

### 4.4 Capsule Acceptance Rules

A capsule is invalid for markets if:

- `zk_genesis_proof` is missing
- proof does not verify
- proof does not bind `pubkey` and `enc_privkey`
- proof version is unknown/deprecated

---

## 5. Bitcoin Anchoring with OP_RETURN

### 5.1 OP_RETURN Definition

OP_RETURN is a Bitcoin Script opcode used to create provably unspendable outputs that can carry a small data payload, without bloating the UTXO set. This protocol uses OP_RETURN strictly for **commitments** (hashes), not content.

### 5.2 Anchor Types

- **Capsule Anchor**: commits to Capsule CID (or its hash/compact form).
- **Market Definition Anchor**: commits to `marketHash`.
- **Witness Set Anchor**: commits to witness set changes.

### 5.3 Payload Size Constraint

OP_RETURN payload must fit standard relay policies (commonly 80 bytes). CIDs or definitions longer than this MUST be compacted using a hash commitment (e.g., SHA256).

---

## 6. Market Definition

A market is a Bitcoin-anchored commitment to a future yes/no question about a capsule’s state.

### 6.1 MarketDefinition Structure

```
MarketDefinition {
  version: u8
  capsuleCIDCommit: bytes32      // commitment to capsule CID (hash/compact)
  predicateType: u8
  predicateParams: bytes
  resolutionHeight: u32
  witnessSetHash: bytes32        // commitment to witness set used for this market
  threshold: u8                  // T of N
  payoutRule: u8                 // e.g., pro-rata
  feePolicy: bytes
}
```

### 6.2 Market Commitment

```
marketHash = SHA256(MarketDefinition)
```

The market is created by broadcasting a Bitcoin transaction containing:

```
OP_RETURN <marketHash>
```

This timestamps the market and makes its rules immutable.

---

## 7. Predicate System (Bitcoin-Native Only)

Predicates MUST be:

- deterministic
- objectively verifiable using Bitcoin data (and protocol-defined anchors)
- binary

### 7.1 Allowed Predicate Types (examples)

1. `OP_RETURN_EXISTS(commitment, beforeHeight)`
2. `TX_CONFIRMED(txid, beforeHeight)`
3. `ADDRESS_BALANCE_GE(scriptPubKey, sats, atHeight)`
4. `UTXO_EXISTS(outpoint, atHeight)`
5. `SUCCESSOR_CAPSULE_ANCHORED(successorCommitment, beforeHeight)`

### 7.2 Predicate Encoding

`predicateParams` is a canonical encoding (e.g., TLV) defined per predicate type.

Rules:
- All encodings MUST be canonical (unique serialization).
- `predicateParams` MUST include any commitments needed to avoid long identifiers.

---

## 8. BTC Staking via Taproot Escrows

### 8.1 Escrow Model

Each market has two Taproot escrows:

- YES escrow
- NO escrow

Participants stake BTC by sending to the escrow corresponding to their position.

### 8.2 Taproot Spend Paths

Each escrow output includes:

1. **Key Path (Cooperative Spend)**  
   Allows cooperative close (refunds, early cancellation) by a pre-committed market operator key or a participant aggregate key, depending on deployment choice.

2. **Script Path YES**  
   Spend authorized only by a valid witness threshold signature over the YES outcome message.

3. **Script Path NO**  
   Spend authorized only by a valid witness threshold signature over the NO outcome message.

### 8.3 Taproot Script Template (high level)

Let `WPK` be the x-only witness aggregate public key.

- YES branch requires Schnorr signature `sigYes` verifying `WPK` over `OutcomeMessage(marketHash, YES, resolutionHeight)`.
- NO branch requires Schnorr signature `sigNo` verifying `WPK` over `OutcomeMessage(marketHash, NO, resolutionHeight)`.

Timeout or refund conditions MAY be included, but MUST be specified in `MarketDefinition` (or referenced by a committed template hash) to prevent ambiguity.

---

## 9. Outcome Attestation Messages

### 9.1 OutcomeMessage

```
OutcomeMessage {
  marketHash: bytes32
  outcome: bool          // YES=1, NO=0
  resolutionHeight: u32
}
```

Witnesses sign:

```
msgHash = SHA256(OutcomeMessage)
```

---

## 10. Decentralized Witness Network (Bitcoin-Only Oracle)

### 10.1 Purpose

Bitcoin cannot enforce “if TX exists then pay YES” inside Script without external attestation. Witnesses provide that attestation.

Witnesses:
- do not custody funds
- do not decide meaning
- only attest facts by signing the deterministic outcome message

### 10.2 Witness Node Requirements

Each witness MUST:

- run Bitcoin Core full node
- index needed data for predicates (OP_RETURN index, tx/utxo lookups)
- evaluate predicates deterministically at `resolutionHeight`
- participate in threshold signing protocol (FROST or MuSig2 threshold)
- post and maintain an on-chain BTC bond

### 10.3 Witness Key Structure

Witnesses collectively control:
- N participants
- threshold T
- one aggregate x-only public key published via `witnessSetHash` commitment

On-chain verification uses the aggregate key.

### 10.4 Threshold Signing Flow

1. When chain tip reaches `resolutionHeight`, witnesses wait K confirmations to reduce reorg risk.
2. Each witness independently evaluates the predicate.
3. Each witness produces a partial signature for the computed outcome.
4. An aggregator (can be any node) collects ≥T partials and produces a final Schnorr signature.

Important: the aggregator is not trusted; invalid aggregation fails verification.

### 10.5 Reorg Handling

Witnesses MUST:
- define K confirmations in their software config
- re-evaluate if reorg occurs before K
- only sign once conditions are stable

---

## 11. Payout Construction and Execution

### 11.1 Winner Spend

After threshold signature exists for the resolved outcome:
- anyone can build a payout transaction spending the winning escrow UTXOs via the corresponding script path
- outputs distribute BTC to winners according to `payoutRule`

### 11.2 Pro-Rata Payout Rule (MVP)

For MVP, payouts are pro-rata among winning side’s escrow contributors.

Implementation note:
- This requires a payout builder that reads stake UTXOs and constructs outputs.
- Fees are deducted per `feePolicy`.

### 11.3 No Admin Resolution

No manual dispute resolution exists. If witness network does not sign, payout cannot occur through outcome paths.

---

## 12. Witness Bonding and Slashing (Bitcoin-Native)

### 12.1 Bond Requirement

Each witness MUST lock BTC in a bond UTXO that is spendable under slashing conditions.

Bond amount and script template are protocol parameters (versioned).

### 12.2 Slashable Offenses

1. **Equivocation**: witness signs both YES and NO for the same `marketHash`.
2. **Contradictory attestations across heights**: witness signs outcomes for the same market with different `resolutionHeight`.
3. **Invalid outcome attestation** (optional in MVP): if a proof can be constructed on-chain that a signature contradicts Bitcoin facts. (This is difficult to enforce purely on-chain; MVP focuses on equivocation.)

### 12.3 Slashing Proof

Equivocation proof consists of:
- two valid Schnorr signatures (or two partials tied to witness identity, depending on scheme) over different outcomes for the same `marketHash` and `resolutionHeight`

### 12.4 Slashing Execution

A slashing transaction spends the witness bond to:
- burn portion to fees, or
- pay to reporter, or
- distribute to honest witnesses

Exact distribution is a protocol parameter.

---

## 13. Witness Set Governance

### 13.1 WitnessSet Definition

Witness set is defined as:

```
WitnessSet {
  version: u8
  members: [pubkey]     // list of witness public keys for threshold scheme
  threshold: u8
  aggregateXOnly: pubkey
  bondParamsCommit: bytes32
}
```

`witnessSetHash = SHA256(WitnessSet)`

### 13.2 Witness Set Anchoring

Witness set changes are published by a Bitcoin transaction containing:

```
OP_RETURN <witnessSetHash>
```

### 13.3 Activation Delay

A witness set update becomes active only after a delay (e.g., D blocks), preventing surprise takeover.

### 13.4 Market-Scoped Witness Sets

Each market binds to a specific `witnessSetHash`. Later witness set changes do not affect existing markets.

---

## 14. Failure Modes and Fallbacks

### 14.1 Witness Liveness Failure

If witnesses do not produce threshold signature:
- escrow cannot be spent via outcome paths
- a committed refund path (timelock) MAY allow refunds after a timeout

Timeout/refund behavior MUST be committed in the market definition or template hash.

### 14.2 Cartel Risk

Witnesses could collude and sign a false outcome. Mitigations include:
- larger N and higher T
- independent operators
- bonding and equivocation slashing
- public auditability and reputation consequences
- market-level witness set selection (choosing a set you trust)

---

## 15. Security Properties

### 15.1 Guaranteed (under standard assumptions)

- BTC-only staking and settlement
- deterministic market definitions anchored on Bitcoin
- public verification of markets and anchors
- no UTXO set pollution from metadata
- capsule origin integrity (assuming ZK + boundary are correct)

### 15.2 Not Guaranteed

- fully trustless resolution (witnesses are still an attestation layer)
- liveness under adversarial witness sets
- privacy against chain surveillance

---

## 16. End-to-End Example

### Market: “Will capsule X be redeemed before block height H?”

**Inputs**
- Capsule CID commitment `C`
- Redemption anchor commitment `R` (OP_RETURN commitment that signals redemption)
- `resolutionHeight = H`

**Predicate**
- `OP_RETURN_EXISTS(R, beforeHeight=H)`

**Market Creation**
1. Build `MarketDefinition` using `C`, predicate, `H`, witness set hash, threshold.
2. Compute `marketHash`.
3. Broadcast TX with `OP_RETURN(marketHash)`.

**Staking**
- Users send BTC to YES escrow or NO escrow.

**Resolution**
- At height H (plus K confirmations), witnesses check if `R` exists in chain before H.
- Witnesses sign YES if it exists else NO.

**Payout**
- Winning escrow is spent using witness threshold signature.
- BTC distributed pro-rata to winners.

---

## 17. Implementation Roadmap (Build Everything)

### Phase 1: Protocol Primitives
- Canonical encodings for MarketDefinition, OutcomeMessage, WitnessSet (with test vectors).
- OP_RETURN anchoring tool (bitcoin-cli + rawtx).
- Capsule validity verifier (includes ZK verification stub/interface).

### Phase 2: Escrow and Payout
- Taproot escrow address generator (YES/NO).
- Payout builder for pro-rata distribution.
- Reference wallet integration to create stake txs.

### Phase 3: Witness Network
- Witness daemon:
  - Bitcoin Core RPC connectivity
  - predicate evaluation engine
  - threshold signing (FROST or MuSig2)
  - reorg handling (K confs)
- Aggregator service (untrusted):
  - collects partials
  - builds final signature
  - publishes outcome artifact

### Phase 4: Bonding and Slashing
- Bond UTXO script templates
- equivocation detection tooling
- slashing transaction constructor

### Phase 5: Governance
- witness set update workflow (anchor + activation delay)
- market registry indexing (optional, off-chain)

### Phase 6: Capsule/Market Integration
- predicate templates tied to capsule events (anchors for redemption, successor capsule, policy changes)
- market creation UI/CLI workflow

---

## 18. Summary

Bitcoin provides:
- time (block height)
- truth (confirmed transactions and UTXOs)
- finality

Taproot provides:
- enforceable escrow and conditional payout paths

Witnesses provide:
- decentralized attestation of deterministic predicates
- threshold accountability, not discretionary control

Capsules provide:
- verifiable objects with provable clean key origin
- public-key visibility with encrypted control

This protocol turns capsule evolution into BTC-denominated prediction markets resolved by Bitcoin facts and enforced by Bitcoin spending conditions.

