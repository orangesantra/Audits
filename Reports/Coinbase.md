# **1L** - A revoked permission can be re-approved via a user using `SpendPermissionManager.sol::approveWithSignature()`.
## Description

In `SpendPermission.sol` contract. A spendpermission revoked by smart account (spendPermission.account) via calling revoke() should not be approved again, without the will of smart account (spendPermission.account). But a revoked function can be approved again via calling approveBatchWithSignature() and this function also not checks wheather the permission is revoked or not.

Revoking is done through this line -

https://github.com/coinbase/spend-permissions/blob/07067168108b83d9662435b056a2a580c08be214/src/SpendPermissionManager.sol#L287

```js
function revoke(SpendPermission calldata spendPermission) external requireSender(spendPermission.account) {
        bytes32 hash = getHash(spendPermission);
        _isRevoked[hash][spendPermission.account] = true;
        emit SpendPermissionRevoked(hash, spendPermission);
    }
```
and approving via signature is done through this line -

https://github.com/coinbase/spend-permissions/blob/07067168108b83d9662435b056a2a580c08be214/src/SpendPermissionManager.sol#L217

```js
function approveWithSignature(SpendPermission calldata spendPermission, bytes calldata signature) external {
        // validate signature over spend permission data, deploying or preparing account if necessary
        if (
            !publicERC6492Validator.isValidSignatureNowAllowSideEffects(
                spendPermission.account, getHash(spendPermission), signature
            )
        ) {
            revert InvalidSignature();
        }

        _approve(spendPermission);
    }
```
So, if the `spendPermission.account` has revoked a permission and approveWithSignature() can be called by anyone and also it doesn't check status of permisssion i.e. wheather it's revoked or not, a revoked permission can be approved again, whithout the will of spendPermission.account as the approveWithSignature() is not using requireSender(spendPermission.spender) modifier.

It will casue gas waste, as there is not point of approving a revoked function in first place and even that without the will of smart account.

## Recommendation
Don't approve a revoked function without smart account's will. Use require check wheather the approval to revoked function is called by spendPermission.account if yes, then proceed and if no, then revert.