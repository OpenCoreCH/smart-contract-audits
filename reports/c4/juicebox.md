# Juicebox

# Minor Issues


- In `addFeedFor` (https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/c363abb67302314c2e061a01f76eb5e5dce2c935/contracts/JBPrices.sol#L115), it would make sens to check if the inverse feed exists (because this one is also queried) to avoid situations where both feeds are added (potentially returning slightly different values) and the result can therefore depend on the order of the currencies
- Like `setTerminalsOf` (https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/69687ba8648b6725764ad941b57cd542964eb64c/contracts/JBDirectory.sol#L258), `setPrimaryTerminalOf` (https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/69687ba8648b6725764ad941b57cd542964eb64c/contracts/JBDirectory.sol#L304) should also be callable by the controller according to the description, but this is not implemented.


# High: Changing a token in the JBTokenStore causes reservedTokenBalanceOf to be wrong

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L216](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L216)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/c363abb67302314c2e061a01f76eb5e5dce2c935/contracts/JBTokenStore.sol#L240](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/c363abb67302314c2e061a01f76eb5e5dce2c935/contracts/JBTokenStore.sol#L240)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L850](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L850)


## Impact
`JBController` uses `JBTokenStore.balanceOf`, which in turn calls `_token.balanceOf` when a token is configured. However, when `JBTokenStore.changeFor` is called (either to remove a project's token or to set it to a new one), `_token.balanceOf` and in turn `JBTokenStore.balanceOf` will change, whereas the internal bookkeeping in `JBController` (`_processedTokenTrackerOf`) will not change. This causes the reserve calculation to return wrong values. Therefore, calls to `_distributeReservedTokensOf` will not distribute the amount of tokens that should be distributed.

## Proof of Concept 
Consider the following diff:
```diff
--- a/test/jb_controller/mint_tokens_of.test.js
+++ b/test/jb_controller/mint_tokens_of.test.js
@@ -107,7 +107,7 @@ describe('JBController::mintTokensOf(...)', function () {
   }
 
   it(`Should mint token if caller is project owner and funding cycle not paused`, async function () {
-    const { projectOwner, beneficiary, jbController } = await setup();
+    const { projectOwner, beneficiary, jbController, mockJbTokenStore } = await setup();
 
     await expect(
       jbController
@@ -132,6 +132,9 @@ describe('JBController::mintTokensOf(...)', function () {
         projectOwner.address,
       );
 
+    // JBTokenStore.changeFor is called, new token has currently supply of 0
+    await mockJbTokenStore.mock.totalSupplyOf.withArgs(PROJECT_ID).returns(0);
+
     let newReservedTokenBalance = await jbController.reservedTokenBalanceOf(
       PROJECT_ID,
       RESERVED_RATE,
```

After the simulated change, `reservedTokenBalance` returns 0 instead of 10,000.

## Recommended Mitigation Steps
Multiple possibilities:
- When the total supply is > 0, add all balances to unclaimed balances
- Do not allow changing tokens when the total supply is > 0, i.e. force burning all tokens first


# Medium: The token in JBTokenStore can be changed to a token that is not owned by the contract

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/c363abb67302314c2e061a01f76eb5e5dce2c935/contracts/JBTokenStore.sol#L267](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/c363abb67302314c2e061a01f76eb5e5dce2c935/contracts/JBTokenStore.sol#L267)


## Impact
In `JBTokenStore.changeFor`, there is the possiblity to change the owner of the old token, but it is neither checked nor automatically enforced (e.g., with an approval of the old owner) that the new token is owned by the token store. If this is not the case, the consequences are severe. Minting and burning will revert (as these functions are restricted to the owner), meaning that the `mintFor` and `burnFrom` functions also will revert.

## Recommended Mitigation Steps
Either check that the token store is the owner or automatically initiate an ownership transfer (where an approval of the previous owner is necessary).


# Medium: When msg.sender is a feeless address, the discount is applied to all splits in _distributeToPayoutSplitsOf

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L838](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L838)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L999](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L999)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1032](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1032)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1098](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1098)


## Impact
The parameter `_feeDiscount` of `_distributeToPayoutSplitsOf` is set to `JBConsts.MAX_FEE_DISCOUNT` if `msg.sender` is a feeless address (line 838). However, this discount is then also applied for allocator sets and transfers to other terminals. I assume that this behavior is not intended, because there is also a check in `_distributeToPayoutSplitsOf` if the allocator set / the terminal (line 1027 / line 1093) is a feeless address. I.e. the recipient should be what deterimes if the address is feeless (which also matches the description "Addresses that can be paid towards from this terminal without incurring a fee."), not the address that initiated the distribution.

Therefore, too little fees are deducted in some scenarios (feeless msg.sender, but for instance an allocator set that is not feeless).

## Recommended Mitigation Steps
Only set `_feeDiscount` to the max value if the fee is 0, do all feeless address checks inside `_distributeToPayoutSplitsOf` based on the recipient.


# Medium: Too much fees deducted in _distributeToPayoutSplitsOf when beneficiary is a feeless address

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1141](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1141)


## Impact
In contrast to transfers to allocator sets and other terminals, it is not checked if the beneficiary of a split is a feeless address.
Therefore, too much fees are deducted when `split.beneficiary` is a feeless address.

## Recommended Mitigation Steps
Add a check if the recipient of this transaction is a feeless address (like in the other two branches).


# Medium: _distributeToPayoutSplitsOf ignores the fact that project owner is a feeless address

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L885](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L885)


## Impact
According to the documentation, `isFeelessAddress` denotes "Addresses that can be paid towards from this terminal without incurring a fee.". However, when the owner of a project is a feeless address, the fee is still deducted from him when paying out the leftover distribution amount, i.e. that is a violation of the specifications and such a project owner should receive the whole leftover amount without any fees.

## Recommended Mitigation Steps
Before calculating `netLeftoverDistributionAmount`, check if the project owner is a feeless address.


# Medium: _useAllowanceOf ignores the fact that beneficiary is a feeless address

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L949](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L949)


## Impact
According to the documentation, `isFeelessAddress` denotes "Addresses that can be paid towards from this terminal without incurring a fee.". However, when the `_beneficiary` within `_useAllowanceOf` is a feeless address, the fee is still deducted from him when paying out the allowance, i.e. that is a violation of the specifications and such a beneficiary should receive the whole amount without any fees.

## Recommended Mitigation Steps
Check if the beneficiary is a feeless address when determining the fee discount.


# Medium: Fees not refunded in addToBalanceOf when isFeelessAddress is updated

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L561](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L561)

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1362](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L1362)


## Impact
When `addToBalanceOf` is called, held fees are only refunded if they aren't originating from a feeless terminal. The motivation for this seems to be that a terminal that got payouts (and paid fees on them) should get them back when transferring the money back, otherwise the fees would be charged multiple times.

However, we do not actually know if the terminal was a feeless terminal at the time the transfer was made, which can lead to situations where fees should be refunded but are not.

## Proof Of Concept
Consider a terminal A that gets a payout from terminal B. At that time, A is not a feeless terminal for B and therefore, a `_heldFeesOf` entry is created. However, B sets `isFeelessAddress[A] = true` afterwards. Some time later, A calls `addToBalanceOf` and does not get back its held fees.  

## Recommended Mitigation Steps
Consider tracking the feeless status at the time of the transfer.


# High: mintTokensOf with a _tokenCount > type(int256).max causes wrong reserve balance when reserve rate is set to MAX_RESERVED_RATE

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L668](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L668)


## Impact
In `mintTokensOf`, the `uint256` value `_tokenCount` is cast to a `int256` when the reserved rate is the `MAX_RESERVED_RATE`. However, this cast will not revert when the value is larger than the maximum `int256` value (but rather overflow and return a negative number), causing wrong values in the `_processedTokenTrackerOf`, meaning the reserved tokens from the mint are lost and the bookkeeping no longer works.

Note that the same problem with the overflow exists in `burnTokensOf`.

## Proof Of Concept
The following diff illustrates the problem
```diff
--- a/test/jb_controller/mint_tokens_of.test.js
+++ b/test/jb_controller/mint_tokens_of.test.js
@@ -16,9 +16,9 @@ import jbTokenStore from '../../artifacts/contracts/JBTokenStore.sol/JBTokenStor
 describe('JBController::mintTokensOf(...)', function () {
   const PROJECT_ID = 1;
   const MEMO = 'Test Memo';
-  const AMOUNT_TO_MINT = 20000;
+  const AMOUNT_TO_MINT = ethers.BigNumber.from("115792089237316195423570985008687907853269984665640564039457584007913129639935");
   const RESERVED_RATE = 5000; // 50%
-  const AMOUNT_TO_RECEIVE = AMOUNT_TO_MINT - (AMOUNT_TO_MINT * RESERVED_RATE) / 10000;
+  const AMOUNT_TO_RECEIVE = AMOUNT_TO_MINT.div(2);
 
   let MINT_INDEX;
 
@@ -476,6 +476,29 @@ describe('JBController::mintTokensOf(...)', function () {
       /*reservedRate=*/ 10000,
     );
 
+    await expect(
+      jbController
+        .connect(projectOwner)
+        .mintTokensOf(
+          PROJECT_ID,
+          1,
+          beneficiary.address,
+          MEMO,
+          /*_preferClaimedTokens=*/ true,
+          /* _useReservedRate=*/ true,
+        ),
+    )
+      .to.emit(jbController, 'MintTokens')
+      .withArgs(
+        beneficiary.address,
+        PROJECT_ID,
+        1,
+        0,
+        MEMO,
+        10000,
+        projectOwner.address,
+      );
+
     await expect(
       jbController
         .connect(projectOwner)
@@ -501,7 +524,7 @@ describe('JBController::mintTokensOf(...)', function () {
 
     let newReservedTokenBalance = await jbController.reservedTokenBalanceOf(PROJECT_ID, 10000);
 
-    expect(newReservedTokenBalance).to.equal(previousReservedTokenBalance.add(AMOUNT_TO_MINT));
+    console.log(newReservedTokenBalance);
   });
 
   it('Should not use a reserved rate even if one is specified if the `_useReservedRate` arg is false', async function () {
```

## Recommended Mitigation Steps
Disallow minting more than `type(int256).max` tokens.


# High: mintTokensOf with a _tokenCount > type(int256).max makes project unusable

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L681](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L681)


## Impact
In `mintTokensOf`, the `uint256` value `_tokenCount` is cast to a `int256` when the reserved rate is 0. However, this cast will not revert when the value is larger than the maximum `int256` value (but rather overflow and return a negative number). This will cause reversals in subsequent calls to `reservedTokenBalanceOf`, making the project unusable.

Note that the same problem with the overflow exists in `burnTokensOf`.

## Proof Of Concept
The following diff illustrates the problem
```diff
--- a/test/jb_controller/mint_tokens_of.test.js
+++ b/test/jb_controller/mint_tokens_of.test.js
@@ -16,9 +16,9 @@ import jbTokenStore from '../../artifacts/contracts/JBTokenStore.sol/JBTokenStor
 describe('JBController::mintTokensOf(...)', function () {
   const PROJECT_ID = 1;
   const MEMO = 'Test Memo';
-  const AMOUNT_TO_MINT = 20000;
+  const AMOUNT_TO_MINT = ethers.BigNumber.from("115792089237316195423570985008687907853269984665640564039457584007913129639935");
   const RESERVED_RATE = 5000; // 50%
-  const AMOUNT_TO_RECEIVE = AMOUNT_TO_MINT - (AMOUNT_TO_MINT * RESERVED_RATE) / 10000;
+  const AMOUNT_TO_RECEIVE = AMOUNT_TO_MINT.div(2);
 
   let MINT_INDEX;
 
@@ -607,6 +607,7 @@ describe('JBController::mintTokensOf(...)', function () {
         projectOwner.address,
       );
 
+    // reservedTokenBalanceOf reverts now, making the system unusable...
     let newReservedTokenBalance = await jbController.reservedTokenBalanceOf(PROJECT_ID, 0);
 
     expect(newReservedTokenBalance).to.equal(previousReservedTokenBalance);
```

## Recommended Mitigation Steps
Disallow minting more than `type(int256).max` tokens.


# High: Total token supply > type(int256).max results in wrong reserves after a migration

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L786](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L786)


## Impact
`int256(tokenStore.totalSupplyOf(_projectId))` in `prepForMigrationOf` will overflow when `tokenStore.totalSupplyOf(_projectId) > type(int256).max`, meaning that `_processedTokenTrackerOf[_projectId]` is set to a wrong (negative) value, causing all reserve calculations after the migration to fail.

## Recommended Mitigation Steps
Cap the maximum supply to `type(int256)` or change the logic for `_processeedTokenTrackerOf` (e.g., a `uint256` array and a separate flag array to indicate if the value is positive).


# High: Reserved tokens not distributed for certain values when migrating

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L817](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L817)


## Impact
In `migrate`, `_processedTokenTrackerOf[_projectId]` (that may be negative) is cast to a `uint256` before the comparison with `tokenStore.totalSupplyOf`. For certain negative values, the casted value will therefore be equal and no reserves will be distributed, although some reserves should be distributed.

## Proof Of Concept
Consider the extreme case `_processedTokenTrackerOf[_projectId] = -1` and `tokenStore.totalSupplyOf(_projectId) = 2^256 - 1`. Then, we will have `uint256(_processedTokenTrackerOf[_projectId]) == tokenStore.totalSupplyOf(_projectId)` and tokens will not be distributed, although they should be.
Of course there are many more combinations (2^256/2 - 1) that cause this behavior.

## Recommended Mitigation Steps
Always distribute reserved tokens when `_processedTokenTrackerOf[_projectId]` is smaller than 0.


# High: Overflow possible in _distributeReservedTokensOf

URLs:

[https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L860](https://github.com/jbx-protocol/juice-contracts-v2-code4rena/blob/733810a0339a5c0cb608345e6fc66a6edeac13cc/contracts/JBController.sol#L860)


## Impact
When `_totalTokens + tokenCount > type(int256).max`, the calculation will overflow and `_processedTokenTrackerOf[_projectId]` will be set to a (wrong) negative value, causing the subsequent reserve calculations to result in wrong values.

## Recommended Mitigation Steps
Cap the tokens that will be minted to `type(int256).max` or change the logic for `_processeedTokenTrackerOf` (e.g., a `uint256` array and a separate flag array to indicate if the value is positive).

