# Deposit-and-Call in Twine  

## 1. Summary

Twine is a cross-VM rollup that allows users to move assets from any source chain (Ethereum, Solana, etc.) and—within the same atomic step—execute arbitrary smart-contract calls on the destination chain.  
The feature is branded **“Deposit-and-Call”** and relies on three pillars:  
• A deterministic event schema (`QueueTransaction`) emitted on L1.  
• A ZSTD-compressed `message` field that carries ABI-encoded calldata.  
• A sequencer that mints the user’s tokens, then replays the decoded calls inside Twine’s EVM.

![](./deposit_and_call.png)

## 2. High Level Architecture

| Layer | Purpose | Key Components |
|---|---|---|
| **L1 Bridge Contracts** | Lock / escrow user funds and emit `QueueTransaction` events |  MessageQueue.sol |
| **L1 Proof Layer** | Produce consensus proofs (block hash, state root, bank hash) | SP1 light-client circuits |
| **Sequencer** | Order & execute L2 blocks; verify L1 proofs | Twine-node |
| **Twine Proof Layer** | Generate execution validity proofs | SP1 `rsp` prover → Groth16 wrapper |
| **Settlement Layer** | Final state commitment | Contracts deployed on Ethereum and Solana |

## 3. Data Models

#### 3.1 Event
```solidity
enum TransactionType {
    Deposit,
    Withdraw,
    Message
}

event QueueTransaction(
    TransactionType   indexed txnType,
    uint64            indexed nonce,
    uint64            chainId,
    uint64            blockNumber,
    address           l1Token,
    address           l2Token,
    address           from,
    address           toTwineAddress,
    uint256           amount,
    bytes             message   // ZSTD-compressed ABI
);
```

#### 3.2 `Message` structure
```solidity
struct ContractCall {
    address targetContract;
    uint64  value;   // ETH value to forward
    bytes   data;    // ABI-encoded calldata
}
```

## 4. End to End flow

#### 4.1 Regular Deposit (baseline)  
1. User calls `bridge.deposit(...)` on L1.  
2. Bridge emits `QueueTransaction(txnType=Deposit, message='')`.  
3. Sequencer notices event, mints wrapped tokens on Twine.  
4. Execution proof is generated & settled; L1 receives confirmation.

#### 4.2 Deposit-and-Call (extended)  
1. **Compression**  
   • User creates an ABI-encoded array of calls based on action they want to execute on twine:  
   ```solidity
   bytes memory payload = abi.encode(
       abi.encodeWithSelector(IERC20.approve.selector, uniswapRouter, amount),
       abi.encodeWithSelector(IUniswapV2Router.swapExactTokensForTokens.selector, <swapParams>)
   );
   bytes memory compressed = ZSTD.compress(payload);
   ```
2. **L1 Submission**  
   • User invokes `bridge.depositAndCall(l1Token, amount, compressed)`  
   • Event emitted: `QueueTransaction(txnType=Deposit, message=compressed, ...)`
3. **Sequencer Pipeline**  
   a. Verify L1 consensus proof.  
   b. Mint `amount` of `l2Token` to `toTwineAddress`.  
   c. Decompress `message` → `ContractCall[]`.  
   d. **Atomic Execution Loop**  
   ```solidity
   require(nonce = messageExecutedCount[chainId] + 1, "not sequential");
   try {
       l2Token.mint(toTwineAddress, amount);
       for (ContractCall calldata call : calls) {
           (bool success, ) = call.targetContract.call{value: call.value}(call.data);
           require(success, "CALL_FAILED");
       }
       l1MessageExecuted[msgHash] = true;
   } catch {
       l1MessageExecuted[msgHash] = false;
   }
   messageExecutedCount[chainId] += 1;
   ```
4. **Proof & Settlement**  
   • SP1 `rsp` circuit generates proof covering token mint + external calls.  
   • Groth16 proof is posted to Ethereum/Solana settlement contracts.  
   • If `l1MessageExecuted[msgHash] == false`, funds can be refunded back on L1
   • Based on `messageExecutedCount[chainId]`, we can see how many L1 messages were executed

### 5. Security Considerations
• **Call Re-entrancy**:  The `call` is done from special executor contract and not from messenger contract, which will not have any funds or roles. So, does not cause reentrancy issue.
• **Gas Limit**: Sequencer enforces a per-call gas cap to avoid DoS.  
• **Decompression**: If the decompressed data cannot be parsed, the entire txn will be marked as not executed.
• **Replay Protection**: `nonce` is globally unique per source chain; mapping `l1MessageExecuted` ensures one-time execution.  


### 6. Optimizations
• ZSTD reduces calldata cost by ~70 % on Ethereum (high zero-byte density).  
• Enables Solana compatibility: compressed payload stays under 1,232-byte tx limit.  
• Batched proofs amortize verification cost across many deposit-and-call requests.

### 7. Failure mode and recovery
| Failure Point | Behavior | Recovery Path |
|---|---|---|
| Compression too large | Revert on L1 | User resubmits with fewer calls |
| Decompression fails | Sequencer marks `l1MessageExecuted=false` | Refund via L1 after timeout with zk proofs |
| External call reverts | Entire tx reverted, mint rolled back | Same as above |

