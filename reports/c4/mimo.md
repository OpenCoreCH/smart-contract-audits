# Mimo Defi

# Medium: minGasReserve of MIMOProxy can be overwritten

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L19](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L19)

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L82](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L82)


## Impact
While there is a check that `owner` is not changed in a `delegatecall`, such a check is missing for `minGasReserve`, which means that the variable can be changed (either maliciously or accidentally because of a storage slot collision). The consequences of this are severe. If the variable is set to a high value, the proxy becomes useless, as `execute` is no longer callable (or only with a very high gas limit) because of an underflow in the `stipend` calculation.

## Proof Of Concept
Bob, the owner of a proxy performs a `delegatecall` via `execute` to a contract that the trusts. However, this contract happens to have a variable at storage slot 2 that is set to 1,0000,000. Therefore, Bob's proxy is unusable after the `execute` call.

## Recommended Mitigation Steps
Mark the variable as `immutable`, as it is unchangeable anyways.


# Medium: Ownership transfer not correctly handled for flash loans

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMOLeverage.sol#L76](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMOLeverage.sol#L76)

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMOEmptyVault.sol#L71](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMOEmptyVault.sol#L71)

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMORebalance.sol#L74](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/MIMORebalance.sol#L74)


## Impact
In response to a flashloan, `proxyRegistry.getCurrentProxy(owner)` is used to retrieve the correct user proxy and check that this proxy has initiated the flash loan. This can be problematic after ownership transfers of the proxy, where `proxyRegistry.getCurrentProxy(owner)` does not reflect the current owner.

## Proof Of Concept
- Alice transfers her proxy to Bob through `transferOwnership`
- Bob now calls `executeAction` on `MimoRebalance` to perform a rebalance. Therefore, `proxyRegistry.getCurrentProxy(address(Bob))` will be called in `executeOperation`, causing the rebalance to fail.

Even worse, when Alice calls `executeAction` through Bob's proxy (e.g., because she gave her permissions to do so before the ownership transfer), it will succeed.

## Recommended Mitigation Steps
Update the proxy registry on ownership transfers.


# Medium: New owner of user proxy can prevent old owner from using the system

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxyRegistry.sol#L50](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxyRegistry.sol#L50)


## Impact
In `deployFor`, `owner()` is called if there is already an entry for the provided address. This can be exploited by a sophisticated attacker to make the system completely unusable for a user.

## Proof Of Concept
- Alice transfers her proxy to Bob through `transferOwnership`
- As Bob can `delegatecall` arbitrary code, he calls another contract that performs a `SELFDESTRUCT` through `execute`.
- Now, when Alice wants to deploy a new user proxy through the registry, `currentProxy.owner()` is called on her previous proxy, which reverts. Therefore, she will never be able to use the system again.

## Recommended Mitigation Steps
Update the proxy registry on ownership transfers.


# High: Automation / management can be set for not yet existing vault

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/automated/MIMOAutoAction.sol#L33](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/automated/MIMOAutoAction.sol#L33)

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/managed/MIMOManagedAction.sol#L35](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/managed/MIMOManagedAction.sol#L35)


## Impact & Proof Of Concept
`vaultOwner` returns zero for a non-existing `vaultId`. Similarly, `proxyRegistry.getCurrentProxy(msg.sender)` returns zero when `msg.sender` has not deployed a proxy yet. Those two facts can be combined to set automation for a vault ID that does not exist yet. When this is done by a user without a proxy, it will succeed, as both `vaultOwner` and `mimoProxy` are `address(0)`, i.e. we have `vaultOwner == mimoProxy`.

The consequences of this are quite severe. As soon as the vault is created, it will be an automated vault (with potentially very high fees). An attacker can exploit this by setting very high fees before the creation of the vault and then performing actions for the automated vault, which leads to a loss of funds for the user.

The same attack is possible for `setManagement`.

## Recommended Mitigation Steps
Do not allow setting automation parameters for non-existing vaults, i.e. check that `vaultOwner != address(0)`.


# Medium: Old owner can still set automation / management for vaults after ownership transfer

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/automated/MIMOAutoAction.sol#L34](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/automated/MIMOAutoAction.sol#L34)

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/managed/MIMOManagedAction.sol#L37](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/actions/managed/MIMOManagedAction.sol#L37)


## Impact
When the ownership of a user proxy is transferred, `proxyRegistry.getCurrentProxy` does not reflect this and still returns the proxy for the previous owner. This can be exploited in the access check of `setAutomation`.
Because `setManagement` has the same check, the same attack is also possible there.

## Proof Of Concept
- Alice transfers her user proxy to Bob.
- Bob can now call `setAutomation` on a vault that is owned by Alice and set a very high fee.
- Afterwards, he can for instance perform a rebalance to get the fee.

## Recommended Mitigation Steps
Either reflect ownership changes in `proxyRegistry.getCurrentProxy` or check in `setAutomation` that `msg.sender` is still the owner of the proxy.


# Medium: Permissions not cleared on user proxy transfer

URLs:

[https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L122](https://github.com/code-423n4/2022-08-mimo/blob/9adf46f2efc61898247c719f2f948b41d5d62bbe/contracts/proxy/MIMOProxy.sol#L122)


## Impact
When the owner of a user proxy is changed, the permissions are not cleared. This can be abused to give oneself permissions and then transfer the proxy to another user.

## Proof Of Concept
- Alice uses `setPermission` to allow herself to call a contract that performs a `SELFDESTRUCT`. 
- Alice transfers her user proxy to Bob.
- Alice can now destroy Bob's proxy, the permissions entry is still set.

## Recommended Mitigation Steps
Remove all permissions on an ownership transfer.