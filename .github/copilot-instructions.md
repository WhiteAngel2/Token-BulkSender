# Token BulkSender - AI Agent Instructions

## Project Overview

**Token BulkSender** is a Solidity smart contract and DAPP that enables efficient batch transfers of ETH or ERC20 tokens to multiple recipients in a single transaction, reducing gas fees through batching (1-200 transfers per tx).

### Core Architecture
- **Single Smart Contract**: `BulkSender.sol` (378 lines) - a Solidity contract handling all batch transfer logic
- **Deployment**: Multi-chain (Mainnet, Ropsten, Rinkeby with different contract addresses)
- **Frontend**: Referenced as https://bulksender.app (web interface for the contract)
- **Key Pattern**: Contract uses `transferFrom()` for ERC20 tokens (requires prior approval), and direct `.send()` for ETH

## Critical Solidity Patterns & Conventions

### VIP System & Fee Management
- **VIP concept**: Users can pay 0 ETH via `registerVIP()` for permanent fee exemption
- **Fee structure**: Non-VIP users pay 0.00 ETH per transaction; VIP users pay 0 ETH
- **Owner privileges**: Only owner can modify VIP lists, fees, and receiver address via `onlyOwner` modifier
- **Key functions**:
  - `registerVIP()` - payable method for users to join VIP
  - `addToVIPList()` / `removeFromVIPList()` - owner bulk management
  - `isVIP()` - view function checking VIP status (owner always VIP)

### Token Transfer Duplicity
The contract exposes **multiple public methods** for the same operation (legacy compatibility):
- **ERC20 same value**: `bulkSendCoinWithSameValue()`, `drop()` (same logic)
- **ERC20 different values**: `bulkSendCoinWithDifferentValue()`, `bulksendToken()` (same logic)
- **ETH same value**: `bulkSendETHWithSameValue()`, `sendEth()` (same logic)
- **ETH different values**: `bulkSendETHWithDifferentValue()`, `bulksend()` (same logic)

**Implementation note**: Internal functions (`coinSendSameValue`, `coinSendDifferentValue`, etc.) contain the actual logic; public methods are thin wrappers.

### Array Handling Constraints
- Maximum array length: **255 addresses per transaction** (enforced: `require(_to.length <= 255)`)
- Optimal batch size: **175 addresses** (tested safe limit to avoid gas failures)
- Recommended max: **200 addresses** (theoretical maximum)
- Arrays must be equal length when using different-value methods: `require(_to.length == _value.length)`

### Fee Logic Edge Cases
When modifying fee calculations, remember:
1. Non-VIP require `msg.value >= sendAmount + txFee` (for ETH) or `msg.value >= txFee` (for ERC20)
2. VIP require only `msg.value >= sendAmount` (no fee surcharge)
3. For ERC20, actual token value comes from `transferFrom()`, ETH value is just gas fee

### SafeMath Library
Custom `SafeMath` utility is embedded (lines 8-38) with overflow protection via require statements. Uses this for all arithmetic operations in the contract.

## Data Flows & Integration Points

### User Approval Workflow (ERC20)
1. User calls `approve(bulkSenderAddress, largeAmount)` on token contract
2. User calls any `bulkSend*()` method with `transferFrom()` calls inside
3. Contract pulls tokens from user via approved allowance
4. **Critical**: Some legacy token contracts require reset approval (FAQ mention) - check if token has strict `approve()` that disallows changing non-zero allowance without resetting to 0 first

### ETH Transfer Pattern
Uses `.send()` for transfers (not `.transfer()` or `.call.value()`). This is **old Solidity style** but acceptable for this use case.

### Receiver Address System
- Default: Falls back to contract owner if `receiverAddress` not set
- Owner can specify alternate receiver via `setReceiverAddress()` to redirect fees/funds
- Used by `getBalance()` to withdraw collected fees and tokens stuck in contract

## Known Limitations & Design Notes

### Solidity Version
Contract uses `pragma solidity ^ 0.4.0` - very old (pre-Solidity 0.5). Do not modernize without comprehensive testing; breaking changes exist in newer versions:
- `constant` keyword deprecated (now `view`/`pure`)
- No explicit `emit` statements in original code
- Integer division behavior differs

### Gas Optimization Opportunities (NOT IMPLEMENTED - document if considering)
- Uses `for` loop with `uint8 i` starting at 1 (skips first recipient)
- Each `send()` call has 2300 gas limit (unsafe for complex fallback functions in recipient)
- Consider if redesigning with lower-level `.call()` pattern

### Missing Best Practices (reflect current design intent)
- No reentrancy guards (but minimal risk: ETH sends are last action, no state changes after)
- No event indexing for filters
- Only owner can withdraw - single point of trust

## Testing & Deployment Considerations

### Network-Specific Addresses
Reference contracts in deployment docs:
- **Mainnet**: `0x2f6321db2461f68676f42f396330a4dc4a8f49df`
- **Ropsten**: `0xfe25a97b5e3257e6e7164ede813c3d4fbb1c2e3b` (now deprecated)
- **Rinkeby**: `0x458b14915e651243acf89c05859a22d5cff976a6` (now deprecated)

### Testing Workflow (from README)
Use Ropsten test network for validation before mainnet. Example data format for CSV/TXT:
```
0x7edbaa86b8d2757322342157d3a44ed0f583514e,12
0x2f6321db2461f68676f42f396330a4dc4a8f49df,1123.126
```

### Proof of Concept
Reference working TX: https://ropsten.etherscan.io/tx/0xd79502721c914f32d42c83d326c62ce635d2df3f012cd6bc667659505e3a4de2

## Common Task Patterns

### Modifying Fees or VIP Logic
1. Edit `setTxFee()` or `setVIPFee()` function - owner-only restrictions already in place
2. Update checks in `ethSendSameValue()`, `ethSendDifferentValue()`, `coinSendSameValue()`, `coinSendDifferentValue()`
3. Remember: ERC20 transfers don't consume ETH from user balance, so VIP exemption only applies to TX fee, not token amounts

### Adding New Transfer Method
- Create thin wrapper public method calling internal function (e.g., `newMethod()` → `ethSendDifferentValue()`)
- Maintain VIP fee logic in the internal function
- Add event emission at end: `emit LogTokenBulkSent(tokenAddr, totalAmount)`

### Batch Managing VIP Addresses
Use `addToVIPList(address[])` or `removeFromVIPList(address[])` for bulk operations rather than individual calls.

## Documentation References
- **User Guide**: See README.md and FAQ.md for use cases and troubleshooting
- **Source Verification**: All networks have Etherscan-verified source code linked in README
- **DAPP Entry Point**: https://bulksender.app
