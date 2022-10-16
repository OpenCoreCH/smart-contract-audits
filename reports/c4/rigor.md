# Rigor

# Minor Issues


- `isPublishedToCommunity` does not check that the `_communityID` exists. It can return true for non-published projects by passing in a `_communityID` of 0. This enables for instance to call `unpublishProject` on unpublished projects (or paying the publish fee for a non-existing project with `_communityID = 0`. While this is not a major issue, it can be confusing (because events are emitted) and building upon this modifier in the future can be dangerous. Consider validating the `_communityID`.
- In `escrow`, it is possible that the `_agent` is the zero address, in which case signature validation succeeds with any invalid signature (i.e., no actual escrow, as there is no agent). Consider adding a check that the `_agent` is non-zero.
- In `Community`, adding members and updating the community hash is possible when the system is changed. As these also change the system state, consider also requiring that the system is not paused.
- There is no way to remove members of a community (e.g., misbehaving members), which might be desirable.
- The comment "// Burn _interestEarned amount wrapped token to lender" is wrong (https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L849), this should be mint instead of burn.
- In general, it is considered good practice to provide a deadline for signatures and a way to revoke them (e.g., when a private key is compromised), which is both currently not implemented.
- In `changeOrder`, it is not checked if the task actually exists. While changing the cost for a non-existing task is not possible (because of the `getAlerts` check), the owner can be set: First, the task will be unapproved, setting the status to inactive. Then, the subcontractor is invited, which succeeds, as the task is inactive. The subcontractor can even accept the invitation, which marks the task as active, although he was never created / initialized. Consider adding a check if the task already exists.
- `raiseDispute` does not include any replay protection, meaning that anyone can raise the same dispute again after one was submitted.
- It is possible to raise disputes for not (yet) existing tasks in `raiseDispute`, which should not be possible
- It is mentioned that "This can be useful when trying to deploy new version of HomeFiProxy". However, there is currently no clean way to do this. When a new HomeFi proxy is deployed and initialized, new proxies are deployed (i.e., the state is lost). Consider adding a way to initialize a new proxy with already existing proxy addresses.
- `SignatureDecoder.recoverKey` does not support [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271), meaning there is no support for smart contracts in all places that use signatures (which are many), which hinders different applications (e.g., building on top of the protocol).
- The number of currencies (3) is hard-coded in different places, consider storing this information in arrays, which enables easy additions of new ones.
- There is no upper limit for the lender fee (https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/HomeFi.sol#L194). Consider enforcing a limit of 1,000 (or even something like 200) to avoid errors and give users an upper limit for the fee.
- `initiateHomeFi` is documented with "Can only be called by HomeFiProxy owner", but this is not true. The function is callable by anyone and sets the owner.
- `recoverTokens` has a hardcoded 3 (https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L369) instead of using the enum value, which can lead to problems when updating the possible enum values.
- `checkPrecision` does not take the number of decimals into account. For USDC with 6 decimals means rounding to 0.1 pennies, whereas the precision is much higher (probably too high, which you want to avoid) for DAI with 18 decimals.



# High: Untyped data signing

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L175](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L175)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L213](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L213)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L530](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L530)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L91](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L91)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L142](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L142)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L167](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L167)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L235](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L235)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L286](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L286)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L346](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L346)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L402](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L402)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L499](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L499)


## Impact & Proof Of Concepts
In many places of the project (see affected code), untyped application data is directly hashed and signed. This is strongly disencouraged, as it enables different attacks (that each could be considered their own issue / vulnerability, but I submitted it as one, as they have all the same root cause):

1.) Signature reuse across different Rigor projects:
While some signature contain the project address, not all do. For instance, `updateProjectHash` only contains a `_hash` and a `_nonce`. Therefore, we can have the following scenario: Bob is the owner of project A and signs / submit `updateProjectHash` with nonce 0 and some hash. Then, a project B that also has Bob as the owner is created. Attacker Charlie can simply take the `_data` and `_signature` that Bob previously submitted to project A and send it to project B. As this project will have a nonce of 0 (fresh created), it will accept it. `updateTaskHash` is also affected by this.
2.) Signature reuse across different chains:
Because the chain ID is not included in the data, all signatures are also valid when the project is launched on a chain with another chain ID. For instance, let's say it is also launched on Polygon. An attacker can now use all of the Ethereum signatures there. Because the Polygon addresses of user's (and potentially contracts, when the nonces for creating are the same) are often identical, there can be situations where the payload is meaningful on both chains.
3.) Signature reuse across Rigor functions:
Some functions accept and decode data / signatures that were intended for other functions. For instance, see this example of providing the data & signature that was intended for `inviteContractor` to `setComplete`:
```diff
diff --git a/test/utils/projectTests.ts b/test/utils/projectTests.ts
index ae9e202..752e01f 100644
--- a/test/utils/projectTests.ts
+++ b/test/utils/projectTests.ts
@@ -441,7 +441,7 @@ export const projectTests = async ({
     }
   });

-  it('should be able to invite contractor', async () => {
+  it.only('should be able to invite contractor', async () => {
     expect(await project.contractor()).to.equal(ethers.constants.AddressZero);
     const data = {
       types: ['address', 'address'],
@@ -452,6 +452,7 @@ export const projectTests = async ({
       signers[1],
     ]);
     const tx = await project.inviteContractor(encodedData, signature);
+    const tx2 = await project.setComplete(encodedData, signature);
     await expect(tx)
       .to.emit(project, 'ContractorInvited')
       .withArgs(signers[1].address);
```
While this reverts because there is no task that corresponds to the address that is signed there, this is not always the case.
4.) Signature reuse from different Ethereum projects & phishing
Because the payload of these signatures is very generic (two addresses, a byte and two uints), there might be situations where a user has already signed data with the same format for a completely different Ethereum application. Furthermore, an attacker could set up a DApp that uses the same format and trick someone into signing the data. Even a very security-conscious owner that has audited the contract of this DApp (that does not have any vulnerabilities and is not malicious, it simply consumes signatures that happen to have the same format) might be willing to sign data for this DApp, as he does not anticipate that this puts his Rigor project in danger.

## Recommended Mitigation Steps
I strongly recommend to follow [EIP-712](https://eips.ethereum.org/EIPS/eip-712) and not implement your own standard / solution. While this also improves the user experience, this topic is very complex and not easy to get right, so it is recommended to use a battle-tested approach that people have thought in detail about. All of the mentioned attacks are not possible with EIP-712:
1.) There is always a domain separator that includes the contract address.
2.) The chain ID is included in the domain separator
3.) There is a type hash (of the function name / parameters)
4.) The domain separator does not allow reuse across different projects, phishing with an innocent DApp is no longer possible (it would be shown to the user that he is signing data for Rigor, which he would off course not do on a different site)


# High: Significant rounding errors for interest calculation

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L686](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L686)


## Impact
The `_noOfDays` calculation in `returnToLender` rounds down to 0. In extreme cases (see Proof Of Concept), this leads to situations where no or only 50% of the interest is paid. These extreme cases are relatively improbable (as it involves steps from the lenders that are generally against his interest, although he might not be aware of that), however situations in between (e.g., only 75% of interest is paid in the end) are possible. This can either happen by accident (a lender often calls `lendToProject` / `reduceDebt` because a project is very fast moving) or because a builder that knows this weakness tricks the lender to call these functions often.
The financial (and reputational damage) can be quite severe, especially for large home construction projects. For instance, if a lender lends $5 million at a 6% APR and 25% of all interest is lost, this is a financial damage of $75,000 for the lender. Moreover, he probably will never use the platform again, as he was promised a 6% APR, but only got 4.5%.

## Proof Of Concept
1.) No interest paid:
Just before one day would be over since `lastTimestamp` was updated the last time, a lender calls `lendToProject` with a very small amount. This causes `claimInterest` to be called and `lastTimeStamp` to be updated, even if no interest was paid out. Because the `_noOfDays` that is used when calling `claimInterest` is zero, no interest is added. When this is repeated every 24 hours (or rather every 24 hours - 10 seconds), interest is *never* accrued. See the following diff for an example:
```diff
diff --git a/test/utils/communityTests.ts b/test/utils/communityTests.ts
index 51dc09e..d1b403f 100644
--- a/test/utils/communityTests.ts
+++ b/test/utils/communityTests.ts
@@ -976,7 +976,24 @@ export const communityTests = async ({
     });
     // project-1
     it('returnToLender(): should accrue interest', async () => {
-      await mineBlock(lendingTimestamp + ONE_DAY);
+      await mineBlock(lendingTimestamp + ONE_DAY - 10);
+      const lender = signers[2].address;
+      const attackLendingPrincipal = 1020;
+      const attackFee = (attackLendingPrincipal * lenderFee) / (lenderFee + 1000);
+      const attackAmountToProject = attackLendingPrincipal - attackFee;
+      // totalLent += amountToProject;
+
+      await mockDAIContract.mock.transferFrom
+        .withArgs(lender, treasury, attackFee)
+        .returns(true);
+      await mockDAIContract.mock.transferFrom
+        .withArgs(lender, projectAddress, attackAmountToProject)
+        .returns(true);
+      const tx = await communityContract
+        .connect(signers[2])
+        .lendToProject(1, projectAddress, attackLendingPrincipal, sampleHash);
+      await tx.wait();
+      await mineBlock(lendingTimestamp + ONE_DAY + 20);
       const numDays = 1;
       const totalInterest = Math.floor(
         (lendingPrincipal * apr * numDays) / 365000,
@@ -988,8 +1005,8 @@ export const communityTests = async ({
         _totalInterest,
         _unclaimedInterest,
       ] = await communityContract.returnToLender(1, projectAddress);
-      expect(_lendingPrincipal).to.equal(lendingPrincipal);
-      expect(_amountToReturn).to.equal(amountToReturn);
+      expect(_lendingPrincipal).to.equal(lendingPrincipal + attackLendingPrincipal);
+      // expect(_amountToReturn).to.equal(amountToReturn);
       expect(_totalInterest).to.equal(totalInterest);
       expect(_unclaimedInterest).to.equal(totalInterest);
     });
```

2.) Only ~50% of interest paid
Alternatively, if the lender calls `reduceDebt` every 1.99 days, we have the following scenario: Interest will be accrued (which causes the `lastTimestamp` to be updated in this case), but only for 1 day, meaning almost 50% of all interest (.99 days every 2 days) is lost.

## Recommended Mitigation Steps
Calculate with a much more detailed granularity (in seconds instead of days).


# High: changeOrder signatures replayable

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L686](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L397)


## Impact
The signatures of `changeOrder` include no nonce or other replay protection mechanism. An attacker can therefore simply take the `_data` and `_signature` of an old transaction and submit it again. While this is not problematic when there is only one `changeOrder` call, it becomes very problematic when there are multiple for the same task.

## Proof Of Concept
Task 1 initially has a cost of 50,000 USD. Then, there is a material shortage, so the builder, contractor and subcontractor agree to change the cost to 70,000 USD (via `changeOrder`). Because the material shortage was not so bad after all, they agree to change it back to 50,000 USD (via `changeOrder`). Now, an attacker can simply resubmit the first call to change it back to 70,000 USD again. Of course, the same can also be done when changing subcontractors.

## Recommended Mitigation Steps
Include replay protection (nonces or storing already processed hashes) for `changeOrder`.


# Medium: Members for not yet existing community can be added

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L184](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L184)


## Impact
When a `_communityID` that does not exist yet is passed to `addMember`, `_community.owner` will be `address(0)`. `SignatureDecoder.recoverKey` (that is used within `checkSignatureValidity`) returns `address(0)` for invalid signatures. Therefore, it is easily possible to add a member to a community that does not exist yet (i.e. with a community ID higher than the current maximum value), as the signature validation succeeds for the owner when an invalid signature is passed. This has two negative consequences:
1.) Members can add itself to a community without the permission of the future owner, which is of course not desired.
2.) As soon as the community is created, the entry in the `members` array is overwritten and `memberCount` is set to 1. However, the `isMember` entry remains `true`. Therefore, we have `membersCount = 1`, `members.length = 1`, but there are two addresses such that `isMember[addr] = true` (the new owner and the previously added member). If multiple (say N) members are added previously, the consequences are even worse, as we will have `membersCount = 1`, `members.length = N` and (N + 1) addresses where `isMember` is true. This breaks invariants of the protocol (that are not fixable by anyone). For instance, it enables situations where an address is not returned in `members(communityID)`, but can still call `publishProject`, as `isMember` is true for the address.

## Recommended Mitigation Steps
Only allow adding members for communities that exist.


# Medium: Expensive task blocks allocation of cheaper ones

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L629](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L629)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L676](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L676)


## Impact
In `allocateFunds`, the loop is exited when there is one task that is more expensive than the current funds. However, there may be other (cheaper) tasks behind this that could be allocated. This decreases the capital efficiency of the protocol significantly.

## Proof Of Concept
For the sake of example let's say that there are 5 tasks with the following costs: [1000, 1, 1, 1, 1]
The last four will only be allocated when the funds available are larger than 1000, although they could be allocated much earlier.

## Recommended Mitigation Steps
Continue looping and reorder the tasks afterwards.


# Medium: Multiple payouts for the same task possible

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L350](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L350)


## Impact & Proof Of Concept
When a task is marked as complete, `Lifecycle.TaskAllocated` remains true. Therefore, we can have the following steps:
- A task is marked as complete via `setComplete`, the sub-contractor is paid.
- `changeOrder` is used to change the sub-contractor, `unApprove` sets `Lifecycle.SCConfirmed` to false and the status to inactive.
- The new sub-contractor accepts the invitation `Lifecycle.SCConfirmed` is now true and the status active again.
- Because `Lifecycle.TaskAllocated` is still true, the call to `setComplete` succeeds.

This is even worse when the change of sub-contractors is A -> B -> A, as the signature that was used to initially pay out A can then be replayed to pay out A again (which could be another issue in itself, that signature replay there is possible, but I did not create a separate one for this).

## Recommended Mitigation Steps
Do not allow changing the sub-contractor for completed tasks. This seems to be intended anyway, as there is a comment `* @dev modifier onlyActive` above `unApprove`, but the modifier is missing.


# Medium: Community does not respect delegation to contractor

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L91](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L91)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L245](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L245)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L536](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L536)


## Impact & Proof Of Concept
A project builder can use `delegateContractor` to (according to the docs) "delegate his authorisation to the contractor.". While this is respected in the `Project` contract, the `Community` contract ignores this delegation completely. All project related activities (either protected by `onlyProjectBuilder` or directly queried in `publishProject` and `escrow`) needs to originate from the builder, even if he delegated his authority.

This can have negative consequences. Let's say a builder goes on a 8-week vacation and delegates his authority to a contractor that he trusts. Because the project does not need as much money as anticipated, the contractor wants to repay the lender to reduce the interest amount. However, this fails, which means that the total cost for the project is unnecessarily increased.

## Recommended Mitigation Steps
Check if the builder has delegated his authority in `Community`, allow the contractor to act on behalf of the builder in this case.


# High: Escrow messages replayable

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L527](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L527)


## Impact & Proof Of Concept
The payload of `escrow` does not include any nonces and there is no other replay protection. Therefore, a call to `escrow` can simply be replayed. If a builder and lender for instance agreed to reduce debt by 1,000 (and the agent agreed to that), one can simply replay the same payload / signatures 4 times more to reduce it by 5,000.

## Recommended Mitigation Steps
Add a nonce or store the already used hashes.


# High: Wrong APR can be used when project is unpublished and published again

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L267](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol#L267)


## Impact
When a project is unpublished from a community, it can still owe money to this community (on which it needs to pay interest according to the specified APR). However, when the project is later published again in this community, the APR can be overwritten and the overwritten APR is used for the calculation of the interest for the old project (when it was unpublished).

## Proof Of Concept
1.) Project A is published in community I with an APR of 3%. The community lends 1,000,000 USD to the project.
2.) Project A is unpublished, the `lentAmount` is still 1,000,000 USD.
3.) During one year, no calls to `repayLender`, `reduceDebt`, or `escrow` happens, i.e. the interest is never added and the `lastTimestamp` not updated.
4.) After one year, the project is published again in the same community. Because the FED raised interest rates, it is specified that the APR should be 5% from now on.
5.) Another 1,000,000 USD is lent to the project by calling `lendToProject`. Now, `claimInterest` is called which calculates the interest of the last year for the first million. However, the function already uses the new APR of 5%, meaning the added interest is 50,000 USD instead of the correct $30,000. 

## Recommended Mitigation Steps
When publishing a project, if the `lentAmount` for the community is non-zero, calculate the interest before updating the APR.


# Medium: raiseDispute may revert erroneously when builder == subcontractor or contractor == subcontractor

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L530](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L530)


## Impact & Proof Of Concept
In `raiseDispute`, when `signer == tasks[_task].subcontractor`, it is checked that the subcontractor has accepted the invitation. However, we can have the following situation:
1.) Bob is contractor for the project and invited as a subcontractor for task 1.
2.) Bob does not want to accept the invitation for the task 1, but he still wants to raise a dispute for the given task. This should be possible according to the specifications, as the builder and contractor should be able to raise disputes for all tasks.
3.) Because we have `signer == tasks[_task].subcontractor` in this scenario and `getAlerts(_task)[2] == false`, `raiseDispute` reverts, although it should not. Therefore, Bob is not able to raise his dispute.

## Recommended Mitigation Steps
Only use `require(getAlerts(_task)[2])` when the last branch of the previous `require` (`signer == tasks[_task].subcontractor`) was really used. If it is true, but `signer == builder` or `signer == contractor` is also true, the invitation does not have to be accepted.


# Medium: changeOrder requires subcontractor signature when the subcontractor address is 0

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L402](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L402)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L485](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L485)


## Impact
Via `changeOrder`, it is possible to set the subcontractor address to 0 (and it is zero when no one is invited). However, when it is updated later again, a signature of the "current subcontractor" (in this case `address(0)`) is still required. This is in contrast to contractors, where the signature is only required when the contractor address is non-zero.

## Proof Of Concept
1.) Task 1 is assigned to the subcontractor Bob.
2.) `changeOrder` with Bob's signature is used to assign task 1 temporarily to address 0 while a new subcontractor is searched.
3.) The price of the task should be changed, which requires the signature of the "current subcontractor" (i.e., `address(0)`)

To be fair, because `SignatureDecoder.recoverKey` returns `address(0)` for invalid signatures, an invalid signature could in theory be submitted in step 3. But I do not assume that this is really intended (for instance, there is also the check in `checkSignatureTask`, although one could simply use an invalid signature when it is `address(0)`) and a design that requires the user to submit invalid signatures in certain scenarios would also be very poor in my opinion.

## Recommended Mitigation Steps
Check if the subcontractor address is zero, do not require a valid signature in such cases.


# Medium: Hash approval not possible when contractor == subcontractor

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L859](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L859)


## Impact & Proof Of Concept
When a contractor (let's say Bob) is also a subcontractor (which can be a valid scenario), it is not possible to use the hash approval feature for `checkSignatureTask`. The first call to `checkSignatureValidity` will already delete `approvedHashes[address(Bob)][_hash]`, the second call therefore fails.

Note that the same situation would also be possible for builder == contractor, or builder == subcontractor, although those situations are probably less likely to occur.

## Recommended Mitigation Steps
Delete the approval only when all checks are done.


# Medium: Finished task can be changed

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L415](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L415)


## Impact
In `changeOrder` when the cost of a task is changed, it is not checked if the task is already finished. If this is the case (in which case `tasks[_taskID].alerts[1]` is true), the consequences are quite severe. When the new cost is lower than the old one, the difference is withdrawn to the builder's account. When the cost is higher, `totalAllocated` is either increased (resulting in wrong values for this variable) or the task is unapproved again and funds are unallocated.

Similarly, the owner can also be changed for a finished task, which sets the task to inactive again. This is especially bad in combinatin with `recoverTokens`, as the function requires all tasks to be finished when recovering funds for the currency of the project.

## Recommended Mitigation Steps
Do not allow any changes for finished tasks.


# Medium: Funds are allocated to inactive tasks

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L474](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L474)


## Impact & Proof Of Concept
`allocateFunds` ignores the status of a task and will therefore be also called on inactive tasks. This reduces the capital efficiency significantly. A task may never become active, for instance because the subcontractor does not accept the invitation. In such a situation, the builder and contractor cannot even reduce the task cost to 0 (to get the allocated funds back), because they do not have the signature of the subcontractor. They need to invite another subcontractor, accept the invitation, and then change the cost to 0 with the signature of this new subcontractor, which is extremely cumbersome.

## Recommended Mitigation Steps
Only allocate funds for active tasks to increate the capital efficiency.


# Medium: Usage of ERC2771 _msgSender() for access control checks

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L46](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L46)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L52](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L52)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/HomeFi.sol#L73](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/HomeFi.sol#L73)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/ProjectFactory.sol#L65](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/ProjectFactory.sol#L65)

[https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/ProjectFactory.sol#L84](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/ProjectFactory.sol#L84)


## Impact
The ERC2771 `_msgSender()` (which extracts the real address in case the message originates from a relayer) is used for access control purposes (e.g., restricting calls only to the admin or another contract of the system) in different places. However, these calls will never go over a relayer. Nevertheless, a compromised relayer will be able to circumvent all access control checks.

Of course, I am aware that the relayer needs to be trusted when using ERC2711. However, security is multi-faceted and one should also try to reduce the implications when an attack happens. Currently, an exploiter of the relayer can take over the whole protocol. If `_msgSender()` would only be used where an end-user interacts with the protocol, the attack surface is smaller and no functionality would be lost with the change.

## Recommended Mitigation Steps
Consider using `msg.sender` for access control checks / whenever only direct calls happen.


# Medium: Multiple parallel disputes with TaskAdd not supported

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L239](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Disputes.sol#L239)

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L238](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L238)


## Impact
The payload of `addTasks` includes a `taskCount` which has to equal the current number of tasks. However, this becomes very problematic when there are multiple disputes in parallel that have `TaskAdd` as their action. This value has to be chosen at the time of the dispute submission, so what should it be? If it is set to the current task count, only the resolution of the first dispute will succeed, for all other ones, the count will be too high. If it is set to the current task count + the sum of all pending `TaskAdd` actions, it assumes that all previous disputes will succeed and will be resolved before this one.

Even if there is only one dispute with this action, another problem is if new tasks are added to the project between the submission and dispute resolution.

## Recommended Mitigation Steps
Use a different technique for the replay protection, e.g. storing the hashes.


# Medium: recoverTokens may always revert

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L368](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L368)


## Impact
In `allocateFunds`, an upper limit for the loop is correctly enforced, as the maximum gas limit could be reached otherwise (when there are too many unallocated tasks). However, the same is not done in `recoverTokens`, where the loop iterates over all tasks when one tries to recover any left-over currency. Therefore, when there are many tasks, it can happen that `recoverTokens` with the currency address always reverts, meaning that the tokens are stuck forever.

## Recommended Mitigation Steps
Enforce an upper limit similarly to `allocateFunds`.


# Medium: recoverTokens does not allow for cancelled tasks

URLs:

[https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L368](https://github.com/code-423n4/2022-08-rigor/blob/f2498c86dbd0e265f82ec76d9ec576442e896a87/contracts/Project.sol#L368)


## Impact
There can be situations where a task remains in status inactive and is never completed (e.g., because no subcontractor accepted it). While the builder & contractor can use `changeOrder` to change the cost to 0, they will still not be able to use `recoverTokens` for retrieving any stuck currency, as the function requires that all tasks (even with cost 0) are finished. Therefore, this money is forever stuck in the contract.

## Recommended Mitigation Steps
Require that the task is either finished or has a cost of 0.
