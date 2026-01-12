# Capsule Model

> Cryptographic containers with verifiable key origin

[< Back to Overview](00-overview.md) | [Next: Bitcoin Anchoring >](02-bitcoin-anchoring.md)

---

## Capsule Identity

A capsule is identified by a **Capsule CID** (content identifier). The CID is the capsule's immutable identity.

---

## Capsule Contents

A capsule MUST include:

| Field | Type | Description |
|-------|------|-------------|
| `pubkey` | Bitcoin public key | x-only or secp256k1 pubkey (plaintext) |
| `enc_privkey` | bytes | Encrypted Bitcoin private key |
| `enc_params` | struct | Encryption metadata (algorithm, KDF params, version) |
| `zk_genesis_proof` | proof | Zero-knowledge proof of sealed key genesis (**required**) |
| `capsule_policy` | optional | Governance/policy metadata (quorum rules, time locks, revocation) |
| `provenance` | optional | Versioning/history references (append-only lineage) |

### Structure Definition

```
Capsule {
  version: u8
  pubkey: bytes33 | bytes32    // secp256k1 compressed or x-only
  enc_privkey: bytes
  enc_params: EncryptionParams
  zk_genesis_proof: ZKProof
  capsule_policy: Option<Policy>
  provenance: Option<ProvenanceChain>
}

EncryptionParams {
  algorithm: u8       // e.g., AES-256-GCM
  kdf: u8             // e.g., Argon2id
  kdf_params: bytes   // Salt, iterations, memory
  version: u8
}
```

---

## Verification vs Control

| Operation | Requirement | Who Can Do It |
|-----------|-------------|---------------|
| **Verification** | `pubkey` only | Anyone |
| **Control** | Decrypt `enc_privkey` | Policy satisfiers only |

- **Verification**: Anyone can derive addresses and verify balances/UTXOs from `pubkey`
- **Control**: Spending is possible only if decryption policy is satisfied; private key is never revealed by the system

---

## Zero-Knowledge Key Genesis (Mandatory)

### The Problem

Encryption does not prevent compromise if a private key is exposed before encryption. A human or process can copy the key, destroying custody integrity permanently.

### The Requirement

Capsules MUST include a **zero-knowledge proof** asserting:

> "I know a private key `k` such that:
> - `pubkey = KeyDerive(k)`
> - `enc_privkey = Encrypt(k, enc_params)`
> - `k` never left a constrained execution boundary
> - `k` was never exported, logged, or observed"

The proof reveals nothing about `k` or encryption randomness.

### Constrained Execution Boundary

Key generation and encryption MUST occur inside an environment that prevents plaintext exfiltration:

| Environment | Trust Model |
|-------------|-------------|
| HSM | Hardware-enforced key isolation |
| Secure enclave (SGX/TDX/SEV) | Hardware attestation + isolation |
| Strict WASM sandbox | No I/O, formal verification |
| Deterministic MPC/threshold ceremony | Multi-party, public transcript |

### Capsule Acceptance Rules

A capsule is **invalid** for markets if:

- [ ] `zk_genesis_proof` is missing
- [ ] Proof does not verify
- [ ] Proof does not bind `pubkey` and `enc_privkey`
- [ ] Proof version is unknown/deprecated

---

## Capsule Lifecycle

```
┌─────────────────┐
│  1. CREATION    │
│  Generate k in  │
│  constrained    │
│  boundary       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. ENCRYPTION  │
│  enc_privkey =  │
│  Encrypt(k)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. PROOF GEN   │
│  ZK proof that  │
│  k never leaked │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. ANCHORING   │
│  OP_RETURN with │
│  capsule CID    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. ACTIVE      │
│  Markets can    │
│  reference this │
│  capsule        │
└─────────────────┘
```

---

## Open Questions

> See [Q1: Cryptographic Origin Guarantees](../review/Q1-zk-genesis.md) for expanded analysis

- Who verifies that a ZK proof actually came from a constrained environment?
- What attestation chain binds proofs to specific hardware/software?
- How do we prevent "valid proof, compromised key" attacks?

---

## Test Vectors

*To be defined in [Appendix: Serialization](../appendix/serialization.md)*
