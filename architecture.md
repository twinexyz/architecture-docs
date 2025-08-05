
# Twine: Multi-Chain ZK-Rollup (Ethereum + Solana)  

## 1. High-Level Architecture

| Layer | Purpose | Components |
|---|---|---|
| **L2 Twine** | General-purpose EVM execution | Centralized sequencer + Execution Layer |
| **Twine Proof Layer** | Generate & aggregate validity proofs | SP1 `rsp` prover, Groth16 wrapper |
| **L1 Proof Layer** | Generate consensus proofs of L1 | Receive verifiable block hash/ state root / bank hash |
| **Settlement Layer** | Final state commitment | Contracts deployed on Ethereum and Solana |
| **Data Availability** | Transaction & proof data | Celestia: store data to reconstruct twine chain |

---

## 2. Execution Flow (End-to-End)

### 2.1 Normal Transaction Flow
1. User sends L2 tx → sequencer **immediately** orders it into next block.  
2. Sequencer **executes** block in the execution layer (reth).  
3. Block data is published to DA layer (calldata / account write).  
4. Full / RPC nodes **replay** block as soon as DA data appears → serve RPC instantly.

### 2.2 Deposit Flow (Cross-L1 Verification)
1. User locks asset on **Ethereum** or **Solana**.  
2. Bridge contract emits `SendToTwine`.  
3. Sequencer **waits for finality** (Ethereum ~12 min, Solana ~32 slots).  
4. Sequencer feeds **finalized block header + Merkle receipt + validator set** to **SP1 light-client guest program**. This is done for solana too with **Account Proofs**
   - Guest outputs a 1-bit attestation: “deposit finalized”.  
5. Sequencer **includes mint txn** in the **next L2 block** **iff** the attestation is valid.  
   - Mint txn is executed like any L2 txn; no extra trust assumptions.
6. Contract variable is updated indicating the l1 message was executed.

> Light-client circuit is part of the same SP1 guest that later proves state transition, so the **mint is provably correct**.

### 2.3 Withdrawal Flow 
1. User burns asset on Twine.  
2. Sequencer waits for 1-block finality on Twine (trivial).  
3. Withdraw prover verifies withdraw transaction actually was executed on twine.
4. Settlement contracts release asset on the **corresponding L1** after proof verification.

---

## 3. Batching & Proof Aggregation

| Parameter | Value |
|---|---|
| Batch Size | TBD |
| Inputs to Batch Prover | `[block_start … block_end]` + `prev_batch_hash` |
| Public Output | `batch_root = MerkleRoot(state_root_start … state_root_end)` <br></br> `batch_hash = keccak(prev_batch_hash ∥ batch_root)` |
| Proof Path | SP1 Groth16 proofs submit to both L1s |

Settlement contracts store only `batch_hash`; individual block states are proven via Merkle proofs against `batch_root`.

---

## 4. Centralized Sequencer

| Responsibility | Notes |
|---|---|
| Transaction ordering | FIFO |
| Deposit/withdrawal attestation | SP1 Verification  |
| Batch proof generation | rsp by succinct |
| L1 submission | Submits single Groth16 proof + DA pointer to both chains |

---

## 5. Node Types

| Node | Duties | Trust Model |
|---|---|---|
| **Full Node** | Downloads DA, replays every block, verifies light-client proofs locally, stores full state. | Trusts DA layer only. |
| **RPC Node** | Same as full node but may prune historical state. | Same. |
| **Light Client** | Downloads DA headers + batch proofs, verifies Merkle proofs against latest `batch_root`. | Trusts DA + ZK proofs. |

All node types can serve **up-to-the-tip** RPC calls even though the **L1 finality proof** lags by one batch.

---

## 6. Security Invariants

1. **No mint without L1 lock**: proven by light-client circuit inside every batch proof.  
2. **No double-spend**: batch proof covers entire state transition; invalid transitions rejected by L1 verifiers.  
3. **Censorship resistance**: users can force l2 transactions from l1 
4. **DA integrity**: any state change must have corresponding data on DA layer → nodes can independently replay.

---


## 7. Quick Reference Diagram

![](./architecture.png)


