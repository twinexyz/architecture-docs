# Twine: Engineering Accomplishments

**Technical Summary for Investors**

---

## Overview

Twine is building trust-minimized multi-chain infrastructure using zero-knowledge cryptography. Our engineering team has delivered production-ready components spanning execution environments, cross-chain relayers, and pioneering consensus provers—establishing Twine as a technical leader in the ZK interoperability space.

---

## Core Infrastructure: Multi-Chain Testnet

We have deployed a **complete multi-chain rollup testnet** built on industry-leading open-source foundations:

- **Execution Layer**: [Reth](https://github.com/paradigmxyz/reth) (Rust Ethereum client) with custom zkBridge precompiles
- **Consensus Layer**: [Lighthouse](https://github.com/sigp/lighthouse) for Ethereum consensus compatibility
- **Settlement**: Multi-chain settlement on Ethereum and Solana simultaneously
- **EVM Compatibility**: Full Solidity support with standard tooling (Hardhat, Foundry, MetaMask)

This testnet validates our core thesis: a single execution environment settling across heterogeneous Layer 1 blockchains with cryptographic verification.

---

## Merkora: Cross-Chain Relayer

**Authored entirely in-house**, Merkora is a production-grade cross-chain relayer written in Rust.

| Aspect | Details |
|--------|---------|
| **Architecture** | 14 modular crates (eth-listener, sol-listener, relayer, consensus, database, rpc, metrics, types, etc.) |
| **Supported Chains** | Ethereum and Solana event streaming |
| **Persistence** | PostgreSQL with SQLx migrations |
| **Observability** | Prometheus metrics, structured logging, configurable retention |
| **Interface** | JSON-RPC server for proof submission and health monitoring |

Merkora handles the full lifecycle: L1 event detection → proof coordination → message relay → L2 state updates.

---

## Reth zkBridge Precompiles

We extended Reth with **custom EVM precompiles** enabling trust-minimized cross-chain operations:

| Precompile | Function |
|------------|----------|
| **Add Native Assets** | Credit ETH/SOL/BTC to Twine accounts via ZK consensus proof |
| **Add Token Assets** | Credit ERC-20/SPL tokens with storage proof verification |
| **Withdraw Native** | Burn assets on Twine with dual-proof verification |
| **Withdraw Token** | Release tokens on L1 after execution + finality proofs |

**Key Innovation**: Dual-proof verification (execution proof + finality proof) ensures withdrawals are cryptographically secured against reorgs and double-spends.

---

## Solana Consensus Prover — Industry First

We implemented the **first functional zero-knowledge Solana light client** on SP1 ZKVM.

| Metric | Value |
|--------|-------|
| **Proving Time** | ~2-5 minutes (batch of 100 slots) |
| **Validator Capacity** | Up to 3,000 validators per proof |
| **Signature Verification** | Ed25519 in zero-knowledge |
| **Consensus Model** | Stake-weighted supermajority (66%+) |
| **Proof Size** | ~256 bytes (Groth16 compressed) |

**Why This Matters**: Prior to Twine, no production-ready ZK Solana light client existed. This enables trust-minimized Solana bridging without oracles or multi-sig committees.

**Technical Features**:
- Rolling hash proof chaining for continuous light client operation
- Tower BFT constraint verification (lockout periods, confirmation depth)
- Stake verification from vote program accounts

---

## Ethereum Consensus Provers on SP1

We built two complementary Ethereum consensus provers with different security/speed tradeoffs:

### Sync Committee Mode (Lightweight)
- **Proving Time**: ~1-2 minutes
- **Validators**: 512 sync committee members
- **Use Case**: Rapid confirmations, low-value transactions

### Full Attestation Mode (Maximum Security)
- **Proving Time**: ~30-45 minutes
- **Capacity**: Up to 65,536 attestations (256×256)
- **Validator Set**: Supports 2M+ validator registry
- **Use Case**: High-value settlements, security-critical operations

**Both modes**:
- BLS12-381 signature aggregation in SP1 ZKVM
- ~300-500k gas on-chain verification
- ~256 byte proof size (Groth16 compressed)

---

## GKR Prototype (R&D)

We have a working prototype of the **GKR (Goldwasser-Kalai-Rothblum) protocol** for optimized consensus proving.

| Current (ZKVM) | Target (GKR) |
|----------------|--------------|
| ~100x overhead vs naive computation | ~10-15x overhead |
| Commits to all intermediate layers | Only commits to inputs/outputs |

**Expected Impact**: 7-10x faster consensus proving, enabling near-real-time cross-chain verification.

---

## Technical Differentiators

| Capability | Twine | Traditional Bridges |
|------------|-------|---------------------|
| **Trust Model** | Zero-knowledge proofs | Multi-sig / oracles |
| **Solana Support** | First ZK light client | Validator committees |
| **Ethereum Support** | Full attestation proofs (65K sigs) | Sync committee only |
| **Settlement** | Multi-chain (ETH + SOL + BTC) | Single chain |
| **Verification** | On-chain cryptographic | Off-chain attestation |

---

## Summary

Our engineering work establishes Twine's technical credibility:

1. **Complete testnet** on Reth + Lighthouse with multi-chain settlement
2. **Merkora**: Production Rust relayer (14 crates, Ethereum + Solana)
3. **zkBridge precompiles**: Trust-minimized asset bridging in EVM
4. **First Solana ZK prover**: Industry-pioneering Ed25519 consensus verification
5. **Ethereum provers**: Sync committee + full attestation modes on SP1
6. **GKR prototype**: 7-10x proving optimization (R&D)

We are building the infrastructure layer for trust-minimized multi-chain interoperability—replacing bridges with mathematics.

---

*For technical deep-dives, see our architecture documentation or contact the team.*

