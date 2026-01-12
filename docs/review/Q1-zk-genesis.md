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

## Status

**In Progress** - Confidential Containers identified as preferred resolution path.

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

## Why Simple Container Checksums Don't Work

A natural intuition is: "Use checksummed containers that spin up, verify they haven't been compromised, and report results."

### What Container Checksums Prove

```
Container image checksum proves:
✅ The code in the container is the expected code
✅ The container has specific configuration (no networking, no volumes)
✅ Anyone can verify the image matches the expected hash

Container checksum does NOT prove:
❌ The container actually ran in isolation
❌ The execution wasn't observed/modified by the host
❌ The key didn't leak through the operator
```

### The Fundamental Problem

**Containers protect the host FROM the container. They don't protect secrets IN the container FROM the host/operator.**

```
Attack scenario with "verified" container:

1. Attacker downloads verified container image (checksum matches ✅)
2. Attacker runs container with debugging/memory inspection enabled
3. Attacker extracts key from container memory mid-execution
4. Attacker reports "checksum verified, here's the proof"
5. Checksum is correct, but key was compromised

The host controls the container runtime.
Root/privileged access defeats container isolation.
```

### What's Missing: Attestation of Execution

Container checksums verify the **image**. What we need is verification of **execution** - proof that specific code ran and produced specific outputs without the operator being able to observe or tamper with intermediate state.

This requires either:
1. Hardware that encrypts execution (TEEs)
2. Multiple independent executors (distributed trust)
3. A trusted third party (notary model)

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

### Option D: Accept Limitation (MVP Bootstrap)

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

### Option E: Confidential Containers (Recommended)

**This combines the intuition of "verified containers" with hardware-backed execution attestation.**

```
Confidential Containers (CoCo) = Checksummed Container + Hardware TEE

Stack:
┌─────────────────────────────────────────┐
│  Key Generation Container (checksummed) │
├─────────────────────────────────────────┤
│  Kata Containers / gVisor runtime       │
├─────────────────────────────────────────┤
│  AMD SEV-SNP / Intel TDX / ARM CCA      │  ← Hardware encrypts memory
├─────────────────────────────────────────┤
│  Host (UNTRUSTED - cannot see inside)   │
└─────────────────────────────────────────┘
```

**How it works:**

1. **Container Image** - Deterministic, reproducible Dockerfile
   - No networking capability
   - No volume mounts
   - Minimal base image
   - Checksum verifiable by anyone

2. **TEE Execution** - Container runs inside hardware-encrypted memory
   - AMD SEV-SNP: Memory encryption + attestation
   - Intel TDX: Similar, Intel ecosystem
   - Host cannot inspect container memory

3. **Attestation Quote** - Hardware signs proof of execution
   ```
   AttestationQuote {
     measurement: SHA384(container_image + runtime_config)
     user_data: SHA256(pubkey || enc_privkey)  // Binds output to quote
     signature: AMD_or_Intel_signed
   }
   ```

4. **Verification** - Anyone can verify
   - Quote signature chains to AMD/Intel root
   - Measurement matches expected container
   - User data matches capsule outputs

**Attestation Flow:**

```
┌──────────────────────────────────────────────────────────────────┐
│                     CAPSULE CREATION                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Load verified container image into TEE                       │
│     └─> measurement = hash(image)                                │
│                                                                   │
│  2. Execute key generation inside TEE                            │
│     └─> k generated (never leaves encrypted memory)              │
│     └─> pubkey = derive(k)                                       │
│     └─> enc_privkey = encrypt(k)                                 │
│                                                                   │
│  3. TEE produces attestation quote                               │
│     └─> Binds measurement + outputs                              │
│     └─> Signed by hardware                                       │
│                                                                   │
│  4. Output capsule                                                │
│     └─> { pubkey, enc_privkey, zk_proof, attestation_quote }     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     CAPSULE VERIFICATION                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Verify ZK proof (pubkey derived from encrypted key)          │
│                                                                   │
│  2. Verify attestation quote signature                           │
│     └─> Chains to AMD/Intel root certificate                     │
│                                                                   │
│  3. Verify measurement matches approved container                 │
│     └─> Container image hash in public registry                  │
│                                                                   │
│  4. Verify quote.user_data binds to capsule outputs              │
│     └─> hash(pubkey || enc_privkey) == quote.user_data           │
│                                                                   │
│  Result: Cryptographic proof that this key was generated         │
│          inside verified code running in hardware isolation      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```rust
struct CapsuleWithConfidentialAttestation {
    // Core capsule data
    pubkey: PublicKey,
    enc_privkey: EncryptedPrivateKey,
    enc_params: EncryptionParams,
    zk_genesis_proof: ZKProof,

    // Confidential Container attestation
    attestation: ConfidentialAttestation,
}

struct ConfidentialAttestation {
    /// The TEE platform (AMD_SEV_SNP, Intel_TDX, etc.)
    platform: TEEPlatform,

    /// Raw attestation quote from hardware
    quote: Vec<u8>,

    /// Expected container image measurement (for verification)
    expected_measurement: [u8; 48],  // SHA384

    /// Certificate chain to platform root
    cert_chain: Vec<Certificate>,
}

enum TEEPlatform {
    AMD_SEV_SNP,
    Intel_TDX,
    ARM_CCA,
}

fn verify_capsule(capsule: &CapsuleWithConfidentialAttestation) -> Result<(), Error> {
    // 1. Verify ZK proof
    verify_zk_genesis_proof(
        &capsule.zk_genesis_proof,
        &capsule.pubkey,
        &capsule.enc_privkey
    )?;

    // 2. Parse and verify attestation quote
    let quote = parse_attestation_quote(
        capsule.attestation.platform,
        &capsule.attestation.quote
    )?;

    // 3. Verify quote signature chains to platform root
    verify_quote_signature(
        &quote,
        &capsule.attestation.cert_chain,
        capsule.attestation.platform
    )?;

    // 4. Verify measurement matches expected container
    if quote.measurement != capsule.attestation.expected_measurement {
        return Err(Error::MeasurementMismatch);
    }

    // 5. Verify quote binds to capsule outputs
    let expected_user_data = sha256(&[
        capsule.pubkey.as_bytes(),
        capsule.enc_privkey.as_bytes()
    ]);
    if quote.user_data != expected_user_data {
        return Err(Error::OutputBindingMismatch);
    }

    Ok(())
}
```

**Container Dockerfile (Reference):**

```dockerfile
# Minimal, reproducible, no-network key generation container
FROM scratch

# Copy statically-linked key generation binary
COPY --from=builder /keygen /keygen

# No networking, no volumes, no capabilities
# (enforced by Kata runtime config)

ENTRYPOINT ["/keygen"]
```

**Pros:**
- Combines container reproducibility with hardware attestation
- More accessible than raw SGX (cloud providers offer CoCo)
- Container tooling is familiar to developers
- Attestation is cryptographically verifiable

**Cons:**
- Still requires TEE hardware (but widely available)
- Trust AMD/Intel/ARM root keys
- TEE vulnerabilities exist (but defense in depth with ZK proof)

**Availability:**
- Azure Confidential Containers (AMD SEV-SNP)
- AWS Nitro Enclaves (different model but similar guarantees)
- GCP Confidential VMs (AMD SEV)
- Kata Containers + AMD SEV (self-hosted)

### Option F: Distributed Container Execution (No Hardware)

If hardware TEEs are unacceptable, distribute trust across multiple executors:

```
Multi-party container execution:

1. Define N independent "capsule notary" operators
2. Capsule creator submits container image to all N
3. Each operator:
   - Verifies image checksum
   - Runs container in isolated environment
   - Returns (pubkey, enc_privkey, execution_log_hash)
4. If all N produce identical output → execution was likely honest
5. Capsule includes attestations from all N operators

Trust model: Collusion of ALL N operators required to cheat
```

**Pros:**
- No hardware dependency
- Scales trust with N
- Operators can be geographically/organizationally diverse

**Cons:**
- Requires N independent trusted parties
- Coordination overhead
- Any single operator COULD cheat (but would be detected if others are honest)

### Option G: Trusted Notary Service (MVP)

Simplest path for MVP - designated trusted parties:

```
Notary model:

1. Protocol defines approved "Capsule Notary" list
2. Notary runs key generation container
3. Notary signs attestation: "I ran image X, got output Y"
4. Capsule includes notary signature
5. Verifiers check notary is on approved list

Upgrade path:
- Start with 1-2 trusted notaries (founders, partners)
- Add more notaries over time
- Eventually require TEE attestation from notaries
```

**Pros:**
- Simple to implement
- Works today with no special infrastructure
- Clear trust model

**Cons:**
- Centralized trust
- Notary compromise = protocol compromise
- Must trust notary operational security

---

## Recommended Resolution Path

| Phase | Approach | Trust Model | Effort |
|-------|----------|-------------|--------|
| **MVP** | Option G (Trusted Notary) | Trust named operator(s) | Low |
| **v1.0** | Option E (Confidential Containers) | Trust TEE hardware | Medium |
| **v2.0** | Option E + F (CoCo + Distribution) | TEE + 1-of-N honest | High |

### MVP Implementation (Option G)

```
1. Define reference container image
   └─> Dockerfile in protocol repo
   └─> Reproducible build with pinned dependencies
   └─> Published image hash

2. Designate initial notary operators
   └─> Protocol founders / trusted partners
   └─> Published notary public keys

3. Notary attestation format
   └─> Binds image_hash + pubkey + timestamp
   └─> Signed by notary key

4. Capsule structure
   └─> { pubkey, enc_privkey, zk_proof, notary_attestation }

5. Verification
   └─> ZK proof valid
   └─> Notary signature valid
   └─> Notary in approved list
   └─> Image hash matches expected
```

### Production Implementation (Option E)

```
1. Same reference container image

2. Confidential Container deployment
   └─> Kata Containers + AMD SEV-SNP
   └─> Or cloud provider CoCo service

3. TEE attestation format
   └─> Hardware quote binding outputs
   └─> Certificate chain to AMD/Intel root

4. Capsule structure
   └─> { pubkey, enc_privkey, zk_proof, tee_attestation }

5. Verification
   └─> ZK proof valid
   └─> TEE quote signature valid
   └─> Measurement matches container
   └─> Output binding verified
```

---

## Acceptance Criteria

This question is **resolved** when:

- [x] Trust model for MVP documented (Trusted Notary)
- [x] Production attestation path identified (Confidential Containers)
- [ ] Reference container Dockerfile created and published
- [ ] Notary attestation format specified
- [ ] TEE attestation verification code implemented
- [ ] Test vectors for valid/invalid capsules exist
- [ ] At least one notary operator committed for MVP

---

## Related

- [Spec: Capsule Model](../spec/01-capsule-model.md)
- [Spec: Security Analysis](../spec/06-security-analysis.md#capsule-key-compromise-pre-encryption)
- [Kata Containers](https://katacontainers.io/) - Container runtime for TEEs
- [AMD SEV-SNP](https://www.amd.com/en/developer/sev.html) - Hardware attestation
- [Confidential Containers](https://github.com/confidential-containers) - CoCo project
