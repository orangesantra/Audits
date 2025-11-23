**H 1** - https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/134#issue-3131506907

**H 2** - https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/344

## **H 1** - Cross-Chain Refund Claim Vulnerability in GatewayCrossChain Contract

## Summary
A high vulnerability exists in the claimRefund() function of the GatewayCrossChain.sol contract that allows any user who knows a valid externalId to claim refunds intended for non-EVM chain users (Bitcoin/Solana), bypassing the authorization checks that protect EVM chain users.

## Root Cause
The vulnerability stems from how the contract handles address validation for non-EVM chain refunds:

https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayCrossChain.sol#L571

address receiver = msg.sender;
```js
if(refundInfo.walletAddress.length == 20) {
    receiver = address(uint160(bytes20(refundInfo.walletAddress)));
}
require(bots[msg.sender] || msg.sender == receiver, "INVALID_CALLER");
```
When `refundInfo.walletAddress.length != 20` (non-EVM addresses like Bitcoin or Solana), receiver remains set to `msg.sender`, making the authorization check `msg.sender == receiver` always true. This effectively bypasses the ownership verification for non-EVM chain refunds.

## Internal Pre-conditions
A cross-chain transaction to a non-EVM chain (Bitcoin/Solana) has failed
The transaction has been logged in the refundInfos mapping with a valid externalId
The walletAddress stored is a non-EVM address (length != 20 bytes)
## External Pre-conditions
Attacker knows a valid externalId for a pending refund, also possible with frontrunning.
Attack Path
Attacker monitors EddyCrossChainRefund events to identify externalId values for non-EVM refunds
Attacker calls claimRefund(externalId) for an identified refund
In the function:
refundInfo is retrieved from storage using the provided externalId
Since `walletAddress.length != 20` (non-EVM address), receiver remains set to msg.sender
The check `msg.sender == receiver` is always true in this case
The tokens are transferred to the attacker instead of the intended recipient
The refund entry is deleted, preventing the legitimate owner from claiming it

## Impact
- Theft of funds intended for Bitcoin or Solana users whose cross-chain transactions failed
- Loss of user trust in the cross-chain protocol
## Mitigation
Update the function with something like this following solutions:

Strict Validation for Non-EVM Refunds:
```js
function claimRefund(bytes32 externalId) external {
    RefundInfo storage refundInfo = refundInfos[externalId];
    
    // Require caller to be an approved bot for non-EVM addresses
    if(refundInfo.walletAddress.length != 20) {
        require(bots[msg.sender], "ONLY_BOTS_CAN_CLAIM_NON_EVM_REFUNDS");
    } else {
        address receiver = address(uint160(bytes20(refundInfo.walletAddress)));
        require(bots[msg.sender] || msg.sender == receiver, "INVALID_CALLER");
    }
    
    require(refundInfo.externalId != "", "REFUND_NOT_EXIST");
    
    // Rest of the function...
}
```

# **H 2** Token Mismatch Vulnerability in Cross-Chain DEX Swap Function 

## Summary
The DODO cross-chain DEX contains a critical vulnerability in the _doMixSwap function of the GatewaySend.sol contract. The function fails to validate that the token specified in the swapData parameter matches the token that was actually transferred to the contract. This allows an attacker to deposit a cheap token but execute a swap using a more valuable token that the contract might already have in its balance, potentially draining valuable assets from the contract.

## Root Cause
The root cause is a lack of input validation in the _doMixSwap function:

https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewaySend.sol#L195
```js
function _doMixSwap(bytes calldata swapData) internal returns (uint256) {
    MixSwapParams memory params = SwapDataHelperLib.decodeCompressedMixSwapParams(swapData);
    
=>   // No validation that params.fromToken matches the token transferred to the contract
    
    if(params.fromToken != _ETH_ADDRESS_) {
        IERC20(params.fromToken).approve(DODOApprove, params.fromTokenAmount);
    }
    
    return IDODORouteProxy(DODORouteProxy).mixSwap{value: msg.value}(
        params.fromToken,
        // ... other parameters
    );
}
```
The function blindly trusts that the token specified in swapData matches the token that was transferred to the contract in the `depositAndCall` function. This missing validation creates a dangerous token mismatch vulnerability.

## Internal Pre-conditions
The GatewaySend.sol contract must hold a non-zero balance of valuable tokens (TokenB) from previous transactions or direct transfers.
## External Pre-conditions
An attacker have access to a less valuable token (TokenA) that can be transferred to the contract
The attacker craft custom swapData with specific routing parameters
## Attack Path
Attacker identifies a valuable token (TokenB) that the contract already holds

Attacker deposits a minimal amount of a less valuable token (TokenA) by calling:
```js
depositAndCall(
    TokenA,               // fromToken - cheap token the attacker provides
    smallAmount,          // amount - minimal amount required
    maliciousSwapData,    // swapData - crafted to use TokenB instead
    targetContract,
    asset,
    dstChainId,
    payload
)
```
Inside depositAndCall, the contract transfers TokenA from the attacker

When `_doMixSwap(swapData)` is called, it decodes parameters showing TokenB as params.fromToken

The contract approves TokenB (not TokenA) for the DODO router

The DODO router executes a swap using the contract's TokenB balance

The resulting tokens are sent cross-chain, effectively stealing the contract's TokenB reserves

## Impact
This vulnerability can lead to:

Theft of valuable tokens from the contract's reserves
Broken cross-chain transactions where users never receive their expected tokens
The severity is HIGH as it allows direct theft of assets with minimal requirements.

## Mitigation
The vulnerability can be fixed by adding explicit validation in the _doMixSwap function to ensure that the token being swapped matches the token that was deposited:
```js
function _doMixSwap(bytes calldata swapData, address expectedFromToken) internal returns (uint256) {
    MixSwapParams memory params = SwapDataHelperLib.decodeCompressedMixSwapParams(swapData);
    
    // Validate that the token in swapData matches the deposited token
    require(params.fromToken == expectedFromToken, "TOKEN_MISMATCH");
    
    if(params.fromToken != _ETH_ADDRESS_) {
        IERC20(params.fromToken).approve(DODOApprove, params.fromTokenAmount);
    }
    
    return IDODORouteProxy(DODORouteProxy).mixSwap{value: msg.value}(
        params.fromToken,
        params.toToken,
        params.fromTokenAmount,
        params.expReturnAmount,
        params.minReturnAmount,
        params.mixAdapters,
        params.mixPairs,
        params.assetTo,
        params.directions,
        params.moreInfo,
        params.feeData,
        params.deadline
    );
}
```
And updating the depositAndCall function to pass the expected token:
```js
function depositAndCall(
    address fromToken,
    uint256 amount,
    bytes calldata swapData,
    address targetContract,
    address asset,
    uint32 dstChainId,
    bytes calldata payload
) public payable {
    // ... existing code ...
    
    // Pass fromToken to _doMixSwap for validation
    uint256 outputAmount = _doMixSwap(swapData, fromToken);
    
    // ... rest of the function ...
}
```