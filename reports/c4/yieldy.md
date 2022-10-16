# Yieldy

# Minor Issues


- The comment in [Line 323](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L323) of `Staking.sol` is wrong, which can be confusing. In the case of `unstakeAllFromTokemak`, we have per definition `balance == _amount` (as the balance of the pool is passed from `unstakeAllFromTokemak`).
- The variable name in [Line 112](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/LiquidityReserve.sol#L112) of `LiquidityReserve.sol` still refers to the old project (FOX).
- Hardcoding the COW contract addresses in [Line 73](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L73) and Line 74 of `Staking.sol` goes against best practices.
- In [Line 88](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L88) of `Staking.sol`, the code from line 84 is duplicated, i.e. the yieldy token approval for the liquidity reserve is executed two times.
- Duplicated code: In [Line 144](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L144) and [Line 372](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L372) of `Staking.sol`, the function `_getTokemakBalance` could be used.
- `amountToRequest` should be passed to `requestWithdrawal` in [Line 326](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L326) of `Staking.sol`, otherwise the wrong value is used when `balance < _amount`.
- In [Line 372](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L372), `tokePoolContract` was already instantiated with `ITokePool`, there is no reason to call `ITokePool(tokePoolContract)` instead of `tokePoolContract.balanceOf`
- In [Line 100](https://github.com/code-423n4/2022-06-yieldy/blob/8400e637d9259b7917bde259a5a2fbbeb5946d45/src/contracts/Yieldy.sol#L100) of `Yieldy.sol`, the new supply is passed as `_previousCirculating` to `_storeRebase`. In [Line 121](https://github.com/code-423n4/2022-06-yieldy/blob/8400e637d9259b7917bde259a5a2fbbeb5946d45/src/contracts/Yieldy.sol#L121), this leads to `totalStakedBefore` and `totalStakedAfter` always being equal (and a wrong `rebasePercent`, which is also calculated with the `_previousCirculating`.
- The `creditAmount` calculation in `Yieldy.sol` is inconsistent. In `transferFrom`, `creditsForTokenBalance` is called, whereas the calculation is done directly in `transfer`, `_mint`, and `_burn`. Consider using one consistent method for that.
- To have a consistent behavior with `rebase`, it should be `require(_totalSupply <= MAX_SUPPLY, ...);` in [Line 257](https://github.com/code-423n4/2022-06-yieldy/blob/8400e637d9259b7917bde259a5a2fbbeb5946d45/src/contracts/Yieldy.sol#L257) of `Yieldy.sol`, i.e. `_totalSupply == MAX_SUPPLY` should also be allowed.


# Medium: coolDown & warmUp period do not work when a low _firstEpochEndTime is passed to initialize

URLs:

- [https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L98](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/Staking.sol#L98)


## Impact
In the constructor of `Staking.sol`, it is not enforced that the `_firstEpochEndTime` is larger than the current `block.timestamp`. If a low value is accidentally passed (or even 0), `rebase` can be called multiple times in sucession, causing the `epoch.number` to increase. Therefore, the coolDown & warmUp period can be circumvented in such a scenario, as `epoch.number >= info.expiry` (in `_isClaimAvailable` and `_isClaimWithdrawAvailable`) will return true after `rebase` caused several increases of `epoch.number`.

## Recommended Mitigation Steps
Either require that `_firstEpochEndTime` is larger than `block.timestamp` or set the expiry of the first epoch to `block.timestamp + _epochDuration`.


# Medium: canBatchContracts fails after a contract was removed

URLs:

- [https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/BatchRequests.sol#L37](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/BatchRequests.sol#L37)
- [https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/BatchRequests.sol#L93](https://github.com/code-423n4/2022-06-yieldy/blob/34774d3f5e9275978621fd20af4fe466d195a88b/src/contracts/BatchRequests.sol#L93)


## Impact
When `contracts[i]` is the 0 address (which happens after `removeAddress` was called), `IStaking(contracts[i]).canBatchTransactions()` will fail for this index, meaning that the whole function will always fail. 

## Recommended Mitigation Steps
Check if the address is equal to 0.

