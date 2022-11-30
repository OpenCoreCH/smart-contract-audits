# Olympus

# Minor Issues

- In `Operator.sol`, when `reserve` is a token with a fee on transfer, the actual amount that is transferred will be too little (https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Operator.sol#L330). Fee-on-transfer tokens are explicitly handled in `BondBaseTeller.sol` (but with a comment stating that these tokens are not supported, which makes the explicit handling a bit weird IMO. If they were supported, the issue would be medium severity in my opinion.
- In `BondBaseCDA`, it is checked that the `payoutTokenDecimals` and `quoteTokenDecimals` are between 6 and 18. While this should not be a problem in OHM itself (as the reserve token and OHM are between these two values), there are tokens with more or less decimals (Gemini Dollar, YAM v2) and the protocol could not be used with them.
- There are some TODOs that should be addressed:
	- `TreasuryCustodian.sol`: Revoking approvals for random addresses possible (https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/TreasuryCustodian.sol#L56)
	- https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Operator.sol#L657
- In `PRICE.sol`, it would make sense in my opinion to bound `numObservations` (https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L97), as the system can become non-functional when the value is too large (because of out of gas errors when iterating over the observations).
- When `Heart` is out of reward tokens, it still tries to transfer them, which will revert (https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Heart.sol#L112). This means that even people that would call `beat()` without a reward (e.g., the OHM team) cannot do so and the reward amount first needs to be set to 0 by the heart admin.  

# Medium: Unexecutable proposals when Actions.MigrateKernel is not last instruction

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/INSTR.sol#L61](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/INSTR.sol#L61)

## Impact & Proof Of Concept
In `INSTR.sol`, it is correctly checked that a `ChangeExecutor` instruction only occurs at the last position to avoid situations where the other instructions are deemed as invalid.
However, the same problem can occur for `MigrateKernel`. For instance, let's say we have a `MigrateKernel` followed by a `DeactivatePolicy` action. The `MigrateKernel` action will change the value of `kernel` within the policy. The `DeactivatePolicy` action tries to call `setActiveStatus` on the policy. However, this has a `onlyKernel` modifier and the call will therefore fail when it is done after the value of `kernel` was changed.

## Recommended Mitigation Steps
Perform the same check for `MigrateKernel`.

# Medium: Activating same Policy multiple times in Kernel possible

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/Kernel.sol#L296](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/Kernel.sol#L296)

## Impact & Proof Of Concept
To check that an already active policy is not added a second time, `isActive()` is called on the policy. However, `policy` could be a malicious contract that always returns `false` for `isActive()`. In such a scenario, it would be possible to activate the policy multiple times for the same Kernel. This would break uniqueness invariants such that `_deactivatePolicy()` no longer works. However, it could also be used for a DoS attack: As `_reconfigurePolicies` and `_migrateKernel` iterate over those lists that now contain duplicates, they could run out of gas if a policy is activated enough times.

## Recommended Mitigation Steps
Check `getPolicyIndex[policy_] != 0` instead of relying on a value of an untrusted contract. 

# Medium: Calculated OHM/RESERVE price may have never existed

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L165](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L165)

## Impact & Proof Of Concept
In `getCurrentPrice()` within `PRICE.sol`, the latest price is used for both the OHM/ETH and RESERVE/ETH price feed. These values are then divided. However, it is never validated that both prices actually have a similar timestamp. Even worse, the allowed time delta for the OHM/ETH is 3-times as large as for the RESERVE/ETH feed.
It is therefore possible that a relatively old price for OHM/ETH is divided by a very recent price for RESERVE/ETH. Depending on the price movements, the reported "current price" will be a price that never existed in the past and never will exist in the future, which off course should not happen.

## Recommended Mitigation Steps
The goal should be to have prices that are as close as possible together, even if that means using no the latest result for one of the feeds because it is better to have a bit older price than one that never existed. One way to accomplish this would be iterating over the last prices and comparing timestamps.

# Medium: No way to increase votes for active proposal

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Governance.sol#L247](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Governance.sol#L247)

## Impact
When a user has already voted for the active proposal, all subsequent calls to `vote` fail. However, it can happen that the user receives new `VOTES` tokens after voting. These tokens cannot be used to vote for the current proposal and are effectively frozen until a new proposal is active. Even worse, because `VOTES.totalSupply()` is considered in `executeProposal()`, it can become much harder (or impossible) to execute a proposal.

## Proof Of Concept
Let's consider an extreme example: User A has 1 `VOTES` token and votes for the active proposal. The `totalSupply()` of `VOTES` at that time is 10. Then, user A gets additional 30 `VOTES` tokens. However, those cannot be used to vote on the active proposal.
For this proposal, it will be impossible to reach the execution threshold, as the maximum number of votes that it can get is 10, but the `totalSupply()` now is 40.

## Recommended Mitigation Steps
Allow increasing the votes for the active proposal.

# Medium: frequency for beat() not always respected

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Heart.sol#L103](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Heart.sol#L103)

## Impact
`lastBeat` is always increased by the frequency and not set to the curernt timestamp. Therefore, if it was not called for a long time, it can be called multiple times in a row which violates frequency. As the moving average is then always updated, this also leads to wrong values for the moving average (instead of a moving average over N * frequency, it can suddenly be a moving average over just a few seconds).

## Proof Of Concept
`lastBeat` is currently at timestamp X and we assume the frequency is 100. `beat()` is not called for 1000 seconds. At timestamp X + 1001, it can now be called ten times in a row and the moving average is updated every time. If these ten updates happen in the same transaction, the moving average is no longer a moving average, but simply the current price at timestamp X + 1001.

## Recommended Mitigation Steps
Set `lastBeat` to the current timestamp.

# Medium: Zero transfer reverts problematic for beat()

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Heart.sol#L112](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/policies/Heart.sol#L112)

## Impact
There are certain tokens (e.g., `LEND`) that revert when the amount is 0. However, `_issueReward` will still try to initiate a transfer, even if the amount is 0. As this function is always called within `beat()`, this can lead to a situation where `beat()` always reverts, which is very problematic (no more price range updates / market operations).

## Proof Of Concept
The reward token is set to `LEND` and the amount to 10. The contract initially calls 1000 `LEND` tokens. After 1000 calls, the reward amount is set to 0, because the contract is out of `LEND`.
However, the OHM team decides to call `beat()` from now on by themselves (and pay for the gas). Because the function always tries a 0 transfer, this fails and calling `beat()` is no longer possible.

## Recommended Mitigation Steps
Only initiate a transfer in `_issueReward` if `reward` is greater than 0 (which is also a good gas optimization).

# Medium: Negative chainlink prices not handled

URLs:

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L167](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L167)

[https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L173](https://github.com/code-423n4/2022-08-olympus/blob/549b96bcf8b97807738572605f6b1e26b33ef411/src/modules/PRICE.sol#L173)

## Impact
Chainlink can return negative values for a price feed (for instance temporary for a short time), as asset prices can be negative (e.g., oil in 2020). Currently, this is not handled correctly and will completely break the system: As the value is directly casted to a `uint256`, this will lead to huge and therefore completely wrong values.

Note that a `reserveEthPrice` of 0 is also not handled explicitly, but will lead to a reversion (which is probably the best thing in such a scenario).

## Proof Of Concept
Chainlink temporarily returns a RESERVE/ETH price of -1. After the casting, `reserveEthPrice` is `type(uint256).max`, meaning that `currentPrice` is 0 (whereas the "true value" for a `reserveEthPrice` that is equal to +1 (i.e., slightly higher) would be `ohmEthPrice * _scaleFactor`).

## Recommended Mitigation Steps
Revert when the returned price is negative to avoid further calculations with completely wrong values.