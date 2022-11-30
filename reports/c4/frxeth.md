# frxETH

# High: Wrong accounting logic when syncRewards() is called within beforeWithdraw makes withdrawals impossible

URLs:

[https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/sfrxETH.sol#L50](https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/sfrxETH.sol#L50)

## Impact
`sfrxETH.beforeWithdraw` first calls the `beforeWithdraw` of `xERC4626`, which decrements `storedTotalAssets` by the given amount. If the timestamp is greater than the `rewardsCycleEnd`, `syncRewards` is called. However, the problem is that the assets have not been transfered out yet, meaning `asset.balanceOf(address(this))` still has the old value. On the other hand, `storedTotalAssets` was already updated. Therefore, the following calculation will be inflated by the amount for which the withdrawal was requested:
```
uint256 nextRewards = asset.balanceOf(address(this)) - storedTotalAssets_ - lastRewardAmount_;
```
This has severe consequences:
1.) During the following reward period, `lastRewardAmount` is too high, which means that too much rewards are paid out too users who want to withdraw. A user could exploit this to steal the assets of other users.
2.) When `syncRewards()` is called the next time, it is possible that the `nextRewards` calculation underflows because `lastRewardAmount > asset.balanceOf(address(this))`. This is very bad because `syncRewards()` will be called in every withdrawal (after the `rewardsCycleEnd`) and none of them will succeed because of the underflow. Depositing more also does not help here, it just increases `asset.balanceOf(address(this))` and `storedTotalAssets` by the same amount, which does not eliminate the underflow.

Note that this bug does not require a malicious user or a targeted attack to surface. It can (and probably will) happen in practice just by normal user interactions with the vault (which is for instance shown in the PoC).

## Proof Of Concept
Consider the following test:
```
function testTotalAssetsAfterWithdraw() public {        
        uint128 deposit = 1 ether;
        uint128 withdraw = 1 ether;
        // Mint frxETH to this testing contract from nothing, for testing
        mintTo(address(this), deposit);

        // Generate some sfrxETH to this testing contract using frxETH
        frxETHtoken.approve(address(sfrxETHtoken), deposit);
        sfrxETHtoken.deposit(deposit, address(this));
        require(sfrxETHtoken.totalAssets() == deposit);

        vm.warp(block.timestamp + 1000);
        // Withdraw frxETH (from sfrxETH) to this testing contract
        sfrxETHtoken.withdraw(withdraw, address(this), address(this));
        vm.warp(block.timestamp + 1000);
        sfrxETHtoken.syncRewards();
        require(sfrxETHtoken.totalAssets() == deposit - withdraw);
    }
```

This is a normal user interaction where a user deposits into the vault, and makes a withdrawal some time later. However, at this point the `syncRewards()` within the `beforeWithdraw` is executed. Because of that, the documented accounting mistake happens and the next call (in fact every call that will be done in the future) to `syncRewards()` reverts with an underflow.

## Recommended Mitigation Steps
Call `syncRewards()` before decrementing `storedTotalAssets`, i.e.:
```
function beforeWithdraw(uint256 assets, uint256 shares) internal override {
	if (block.timestamp >= rewardsCycleEnd) { syncRewards(); }
	super.beforeWithdraw(assets, shares); // call xERC4626's beforeWithdraw AFTER
}
```
Then, `asset.balanceOf(address(this))` and `storedTotalAssets` are still in sync within `syncRewards()`.

# Medium: sfrxETH.depositWithSignature / sfrxETH.mintWithSignature calls can be front-run, causing them to fail

URLs:

[https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/sfrxETH.sol#L50](https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/sfrxETH.sol#L50)

## Impact
An attacker that observes calls to `sfrxETH.depositWithSignature` / `sfrxETH.mintWithSignature` in the mempool can use the provided signature to call the `permit` function himself. Because the nonce will be increased, the deposit call will then fail. This can therefore be exploited to continously prevent users from depositing into the vault.

## Proof Of Concept

- Bob wants to deposit 1 frxETH into the vault by calling `sfrxETH.depositWithSignature` and providing the signature which is used for `permit`.
- The attacker Charlie sees this transaction in the mempool, takes the signature, and calls `frxETH.permit(address(bob), address(sfrxETH), amount, deadline, v, r, s)`. This call suceeds and increases Bob's nonce by 1.
- Because of the increased nonce, `asset.permit(msg.sender, address(this), amount, deadline, v, r, s)` (within `depositWithSignature`) will revert. Bob is confused that the call reverted, but he sees that no frxETH were transferred out, so he thinks it is save to try it again.
- When Charlie sees the next call, he can repeat the front-running. Like that, Bob is never able to succesfully deposit into the vault.

## Recommended Mitigation Steps
The function should check if the approval amount is already equal to the provided data. In such a situation, it should not call `permit`.


# High: frxETHMinter.depositEther may run out of gas, leading to lost ETH

URLs:

[https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/frxETHMinter.sol#L129](https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/frxETHMinter.sol#L129)

## Impact
`frxETHMinter.depositEther` always iterates over all deposits that are possible with the current balance (`(address(this).balance - currentWithheldETH) / DEPOSIT_SIZE`). However, when a lot of ETH was deposited into the contract / it was not called in a long time, this loop can reach the gas limit. When this happens, no more calls to `depositEther` are possible, as it will always run out of gas.

Of course, the probability that such a situation arises depends on the price of ETH. For >$1,000 it would need require someone to deposit a large amount of money (which can also happen, there are whales with thousands of ETH, so if one of them would decide to use frxETH, the problem can arise). For lower prices, it can happen even for small (in dollar terms) deposits. And in general, the correct functionality of a protocol should not depend on the price of ETH.

## Proof Of Concept
Jerome Powell continues to rise interest rates, he just announced the next rate hike to 450%. The crypto market crashes, ETH is at $1. Bob buys 100,000 ETH for $100,000 and deposits them into `frxETHMinter`. Because of this deposit, `numDeposit` within `depositEther` is equal to 3125. Therefore, every call to the function runs out of gas and it is not possible to deposit this ETH into the deposit contract.

## Recommended Mitigation Steps
It should be possible to specify an upper limit for the number of deposits such that progress is possible, even when a lot of ETH was deposited into the contract.

# Medium: frxETHMinter: Non-conforming ERC20 tokens not recoverable

URLs:

[https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/frxETHMinter.sol#L200](https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/frxETHMinter.sol#L200)

## Impact
There is a function `recoverERC20` to rescue any ERC20 tokens that were accidentally sent to the contract. However, there are tokens that do not return a value on success, which will cause the call to revert, even when the transfer would have been succesful. This means that those tokens will be stuck forever and not recoverable.

## Proof Of Concept
Someone accidentally transfers USDT, one of the most commonly used ERC20 tokens, to the contract. Because USDT's transfer [does not return a boolean](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code), it will not be possible to recover those tokens and they will be stuck forever.

## Recommended Mitigation Steps
Use OpenZeppelin's `safeTransfer`.

# Medium: ERC20PermitPermissionedMint: Minter may not be deletable

URLs:

[https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/ERC20/ERC20PermitPermissionedMint.sol#L84](https://github.com/code-423n4/2022-09-frax/blob/dc6684f77b4e9bd965e8862be7f5fb71473a4c4c/src/ERC20/ERC20PermitPermissionedMint.sol#L84)

## Impact
No elements are ever deleted in `minters_array` within `ERC20PermitPermissionedMint`, they are only set to `address(0)`. When there are a lot of minters, it is therefore possible that this array will grow too large at one point and all iterations over it will run out of gas. The consequences of this would be quite severe. It would no longer be possible for governance to call `removeMinter`, as this function iterates over the whole array and therefore would always run out of gas. A malicious minter could therefore no longer be removed.

## Proof Of Concept
There were a lot of fluctuations for the minters during the first 5 years of frxETH because `frxETHMinter` was redeployed a lot and the design was changed such that there are multiple minters with different addresses. In total, 600 minters were added, but 550 were deleted again. Because of this, `minters_array` contains 600 entries, but 550 of them are `address(0)`. A new minter contract is added, but it turns out that the contract has a critical vulnerability, allowing anyone to mint tokens for free. Therefore, governance decides to remove the minter in an emergency proposal. Because of the position of this minter, the call always returns out of gas and removal does not succeed.

## Recommended Mitigation Steps
In my opinion, there is no reason to keep the indices within the array the same. Therefore, you could also completely delete an element (swap it with the last array element, `pop` the array) instead of setting the addresses to zero.