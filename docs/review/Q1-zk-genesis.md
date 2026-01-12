# Q1: ZK Genesis Proof Trust Boundary

> What actually prevents a compromised key from having a valid proof?

[< Back to Review Index](README.md)

---

## Summary

The protocol requires ZK proofs of "sealed key genesis" but doesn't specify how verifiers confirm that proofs actually originated from constrained execution environments (HSM, enclave, WASM sandbox, MPC).

---

## Severity

**Critical**

## MVP Blocker

**No** - Can proceed with documented trust assumptions.

---

## The Question

Section 4.3 of the spec lists acceptable constrained execution boundaries:
- HSM
- Secure enclave (SGX/TDX/SEV)
- Strict WASM sandbox with no I/O
- Deterministic MPC/threshold ceremony

**But who verifies that a given ZK proof actually came from one of these environments?**

If someone generates a keypair normally, then later constructs a proof claiming it came from a "strict WASM sandbox" — what prevents this? Is there an attestation chain back to specific hardware/software, or does the protocol trust the proof system's circuit to encode these constraints?

---

## Why This Matters

The entire "tainted capsule" prevention model depends on this proof being unforgeable. If the proof merely asserts "key was generated correctly" without binding to verifiable execution context, the security degrades to:

> "Trust whoever created this capsule didn't copy the key first"

This is the exact problem the ZK requirement is supposed to solve.

### Attack Vector

```
Attacker's workflow:
1. Generate keypair (k, pubkey) normally, with k exposed
2. Copy k to attacker's storage
3. Encrypt k to produce enc_privkey
4. Generate ZK proof claiming "k never left sandbox"
5. Publish capsule with valid proof but compromised key

Result: Proof verifies, but key integrity is violated
```

---

## Technical Analysis

### What ZK Proofs Can Prove

A ZK proof demonstrates knowledge of a witness satisfying a relation:

```
∃ w : R(x, w) = true

Where:
  x = public input (pubkey, enc_privkey)
  w = private witness (k, encryption randomness)
  R = relation (key derivation + encryption)
```

The proof shows:
- ✅ Prover knows `k` such that `pubkey = f(k)`
- ✅ Prover knows randomness such that `enc_privkey = E(k, r)`
- ❌ Does NOT prove WHERE `k` was computed
- ❌ Does NOT prove `k` wasn't copied before encryption

### The Gap

ZK circuit can encode computational constraints, but cannot prove **execution environment properties** without additional binding.

---

## Proposed Resolutions

### Option A: Hardware Attestation (SGX/TDX/SEV)

Require attestation quote from hardware enclave:

```rust
struct CapsuleWithAttestation {
    pubkey: PublicKey,
    enc_privkey: EncryptedPrivateKey,
    zk_genesis_proof: ZKProof,
    attestation_quote: AttestationQuote,  // NEW
    expected_measurement: Hash,            // NEW
}
```

**Verification:**
1. Verify ZK proof
2. Verify attestation quote signature (Intel/AMD CA chain)
3. Verify quote measurement matches expected enclave code
4. Verify quote commits to pubkey

**Pros:**
- Hardware-enforced isolation
- Attestation chain to hardware vendor

**Cons:**
- Requires specific hardware (Intel SGX, AMD SEV)
- SGX has known side-channel vulnerabilities
- Trust Intel/AMD root keys
- Enclave code must be auditable

### Option B: MPC Ceremony with Public Transcript

Multi-party key generation with published transcript:

```
Ceremony:
1. N participants run threshold key generation
2. Randomness from public beacon (Bitcoin block hash)
3. All messages published to transcript
4. Final pubkey = combined output
5. enc_privkey = threshold-encrypted shares
```

**Verification:**
- Replay transcript to verify correct execution
- Verify randomness source
- Verify all N participant signatures

**Pros:**
- No hardware dependency
- Publicly auditable
- Security from N-1 honest participants

**Cons:**
- Coordination overhead
- Requires N independent parties
- Still trusts ceremony software

### Option C: Proof-Carrying Code (RISC Zero / SP1)

Generate execution proof for WASM module:

```
1. Define reference WASM module with no I/O
2. Compile with zkVM proving backend
3. Generate proof: "I ran module M and got output (pubkey, enc_privkey)"
4. Proof includes commitment to module hash
```

**Pros:**
- Proves specific code ran
- No hardware dependency
- Formally verifiable module

**Cons:**
- Proof size and verification cost
- Defining "no I/O" formally is complex
- zkVM soundness assumption

### Option D: Accept Limitation (MVP)

Document the trust model explicitly:

```
MVP Capsule Acceptance:
- ZK proof MUST verify
- Proof circuit specification MUST be public
- Capsule creator identity SHOULD be disclosed
- Trust assumption: "Creator followed clean generation procedure"
```

**Pros:**
- Simple, no infrastructure requirements
- Honest about trust model

**Cons:**
- No actual enforcement
- Relies on reputation and legal consequences

---

## Recommended Approach

**Phase 1 (MVP):** Option D - Accept limitation with disclosure

```
Acceptance criteria:
- [ ] ZK proof verifies
- [ ] Creator identity disclosed
- [ ] Explicit trust assumption documented
```

**Phase 2 (Production):** Option A or B

```
Acceptance criteria:
- [ ] Attestation chain present
- [ ] Attestation binds to pubkey
- [ ] Multiple attestation types supported
```

**Phase 3 (Hardened):** Combination approach

```
Acceptance criteria:
- [ ] Hardware attestation OR MPC ceremony
- [ ] Formally verified generation code
- [ ] Public registry of approved methods
```

---

## Acceptance Criteria

This question is resolved when:

- [ ] Trust model for MVP is documented in spec
- [ ] Phase 2 attestation requirements defined
- [ ] Verification code for at least one attestation type implemented
- [ ] Test vectors for valid/invalid capsules exist

---

## Related

- [Spec: Capsule Model](../spec/01-capsule-model.md)
- [Spec: Security Analysis](../spec/06-security-analysis.md#capsule-key-compromise-pre-encryption)
