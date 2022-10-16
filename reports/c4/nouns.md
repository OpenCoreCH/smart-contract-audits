# Nouns DAO

# Minor Issues


- It would make more sense to me if the votes need to be equal to or higher than the proposalThreshold (https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L198). This would also be in line with the error message. Because it currently states "proposer votes below proposal threshold", but also reverts when the votes are equal to the threshold.
-  There is no way to revoke a signature that was given for delegation. Many projects have such a function, e.g. when the signature was provided by accident or to the wrong person.
- In `_delegate`, the event `DelegateChanged` is emitted, even when the delegatee was not changed. This could be confusing for applications that consume these events.
- `add96` (https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/base/ERC721Checkpointable.sol#L269) will never show the `errorMessage`, because the calculation already reverts with Solidity 0.8. Consider using an `unchecked` such that the correct message is shown when an overflow occurs.
- Since Solidity 0.8, `block.chainid` could be used, there is no need to use inline assembly to retrieve the chain ID.
- The comment for `MAX_QUORUM_VOTES_BPS_UPPER_BOUND`(https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L86) is wrong, as it states 4,000 basis points, but the value is 6,000 basis points.
- In `_burnVetoPower`, the comment `// Check caller is pendingAdmin and pendingAdmin ≠ address(0)` is wrong (probably a copy paste error).
- `state` returns `ProposalState.Defeated` for the (non-existing) proposal with ID 0. While this is not extremely problematic, it is wrong, could lead to problems in the future, and it is not in line with the function otherwise (as it reverts for non-exiting proposals). Consider reverting for ID 0.
- Unlike the transfer of the admin, the transfer of the vetoer is not a two-step process with confirmation. Consider also implementing a two-step process to avoid accidentally losing the vetoer possibility when it is not intended.


# Medium: Delegation to zero address via delegateBySig possible

URLs:

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/base/ERC721Checkpointable.sol#L143](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/base/ERC721Checkpointable.sol#L143)


## Impact
In `delegateBySig`, it is possible to delegate to `address(0)`. In such cases, the votes are burned in `_moveDelegates` (the `srcRep` is updated, but the `dstRep` is not). The account of the delegator becomes useless. Delegating to other addresses is no longer possible and transferring the NFTs also becomes impossible.

## Proof Of Concept
Alice has 2 tokens that are delegated to Bob. She calls `delegateBySig` with `address(0)` as delegatee. Bob's balance is decreased by two as a result.
If Alice now calls `delegate` or `delegateBySig` again, there is always an underflow, as `delegates(address(Alice)) = Alice` and `_moveDelegates` therefore tries to subtract from Alice's balance. The same happens when she tries to transfer the NFTs because of the `_moveDelegates(delegates(from), delegates(to), 1);` within `_beforeTokenTransfer`.

## Recommended Mitigation Steps
Do not allow delegation to the zero address.


# Medium: ETH cannot be locked for specific proposal

URLs:

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L333](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L333)


## Impact
A proposal may have a `value` set that is sent in `NounsDAOExecutor` when the call is executed. However, the ETH balance of the `NounsDAOExecutor` is used when doing so. There is no way to lock / reserve ETH for a specific proposal, which can lead to erroneous behavior.

## Proof Of Concept
Alice proposes to transfer 1 ETH to contract F. There are a lot of votes in favor of this proposal, so Alice (or the Nouns DAO) decides to deposit 1 ETH into the Executor contract. However, there is also a successful proposal from Bob which transfer 1 ETH to contract D. This proposal already succeeded some weeks ago, but there was not enough ETH to execute it. Now, that the executor contract has 1 ETH, `execute` can be called on Bob's proposal and Alice's ETH is used for a completely different purpose.

## Recommended Mitigation Steps
A relatively simple fix would be to make execute `payable` and forward the ETH if `msg.value` is greater than 0. Like that, ETH could be associated with a single proposal only by sending the corresponding amount when executing.


# Medium: Two identical calls in one proposal not possible

URLs:

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L313](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L313)


## Impact
Because `queueOrRevertInternal` reverts when the same call (same target, value, data, signature, eta) is already queued, it is not possible to have the same call two times in the same proposal, which can be limiting for some applications.

## Proof Of Concept
There is an auction contract for a token with a `buy()` function. Because the function does not accept an amount parameter, the user has to call it multiple times when he wants to buy multiple tokens.
Alice creates a proposal to buy two of those tokens, i.e. a proposal that has two times exactly the same call. The proposal succeeds and she wants to queue it. However, this reverts and the Nouns DAO does not get this awesome new token.

## Recommended Mitigation Steps
Allow the same call multiple times (in the same proposal) or introduce a `times` parameter to indicate how many times the contract should be called.


# Medium: Rounding down in quorum calculation

URLs:

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L912](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L912)

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L1007](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L1007)


## Impact
`bps2Uint` that is used within the quorum calculation rounds down. While this is no problem when the number of decimals is large (the difference between a required quorum of 18,000,000 or 18,000,001 is negligible), it can be problematic for a project like Nouns that has 0 decimals and not too many tokens.

## Proof Of Concept
Let's say the total supply is 49 tokens and the `minQuorumVotesBPS` is set to `400`, i.e. 4%. Because of the rounding down, 1 single vote for the proposal (when there are no votes against it) is sufficient, although the "correct" quorum would be 1.96
If the function would round up instead, 2 votes would be necessary, which would be more correct in this situation.

## Recommended Mitigation Steps
Consider rounding up in the quorum calculation.


# Medium: System will stop working in February 2106

URLs:

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/base/ERC721Checkpointable.sol#L239](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/base/ERC721Checkpointable.sol#L239)

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L923](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L923)

[https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L965](https://github.com/code-423n4/2022-08-nounsdao/blob/c1c7c6201d0247f92472419ff657b570f9104565/contracts/governance/NounsDAOLogicV2.sol#L965)


## Impact
In February 2106, `block.timestamp` will be larger than `type(uint32).max`, which means that the casting to a `uint32` that is performed in multiple places will will no longer work. Because these values are stored within the checkpoints, an update before that will not be straightforward.

*Rationale for medium risk:* The same issue was classified as medium risk [in a previous contest](https://code4rena.com/reports/2022-06-nibbl#m-02-twavsol_gettwav-will-revert-when-timestamp--4294967296).

## Recommended Mitigation Steps
Use a `uint64` for the timestamp.

