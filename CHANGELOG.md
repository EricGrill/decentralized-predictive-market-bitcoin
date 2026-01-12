# Changelog

All notable changes to this protocol specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added
- STAKE_REGISTRATION anchor type (0x05) for on-chain winner identification
- Self-contained market anchors - full MarketDefinition can be embedded on-chain
- Stake registration workflow in Bitcoin anchoring spec

### Changed
- **BREAKING**: Updated for Bitcoin Core v30 - OP_RETURN limit removed (now 100KB)
- Bitcoin anchoring spec rewritten to leverage unlimited OP_RETURN
- Q4 (Winner Identification) resolved via on-chain OP_RETURN approach
- Q1 (ZK Genesis) updated with Confidential Containers resolution path

### Fixed
- Q4 no longer MVP blocker - on-chain solution eliminates coordinator trust

---

## [0.1.0-draft] - 2026-01-12

### Added

#### Protocol Specification
- Initial protocol overview with goals and non-goals
- Capsule model with ZK genesis proof requirements
- Bitcoin anchoring via OP_RETURN commitments
- Market definition structure with predicate system
- Taproot escrow system for YES/NO outcomes
- Witness network specification with FROST threshold signing
- Bonding and slashing mechanisms
- Security analysis with trust model

#### Documentation Structure
- Modular spec documents (`docs/spec/`)
- Design review questions (`docs/review/`)
- Technical appendices (`docs/appendix/`)
- GitHub issue templates

#### Design Review
- Q1: ZK Genesis Proof Trust Boundary
- Q2: Witness Economic Model
- Q3: UTXO Stability During Resolution
- Q4: Winner Identification (MVP Blocker)
- Q5: Information Asymmetry

### Known Issues
- Q4 (Winner Identification) must be resolved before implementation
- Witness economic model requires detailed specification
- ZK attestation chain not fully specified

---

## Version History

| Version | Date | Status |
|---------|------|--------|
| 0.1.0-draft | 2026-01-12 | Current |

---

## Versioning Policy

### Specification Versions

- **Major** (X.0.0): Breaking changes to protocol or wire format
- **Minor** (0.X.0): New features, backward compatible
- **Patch** (0.0.X): Clarifications, typo fixes, non-breaking improvements

### Draft vs Release

- **Draft** (`-draft`): Under active development, may change
- **RC** (`-rc.N`): Release candidate, feature complete
- **Release** (no suffix): Stable, implementation-ready

---

## Migration Notes

*No migrations yet - this is the initial version.*

---

## Related

- [Protocol Specification](docs/README.md)
- [Design Review](docs/review/README.md)
- [Contributing](CONTRIBUTING.md)
