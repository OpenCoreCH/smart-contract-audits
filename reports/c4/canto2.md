# Canto 2 

# Minor Issues

- Existing proposals can be overwritten in the function `AddProposal` (https://github.com/Plex-Engineer/manifest-v2/blob/3aedf21400a470a50c9be6e38f1ca57f061eca46/contracts/Proposal-Store.sol#L46). This is probably not desired, as an approved proposal that is added to the store should be immutable. Consider checking if the proposal exists in the function.
- [This comment](https://github.com/Plex-Engineer/manifest-v2/blob/3aedf21400a470a50c9be6e38f1ca57f061eca46/contracts/Proposal-Store.sol#L6) in `Proposal-Store.sol` is wrong and seems to be copied from `ERC20DirectBalanceManipulation.sol`.
- In `NoteInterest.sol`, [it is mentioned](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L42-L43) that `adjusterCoefficient` has a default value of 1, although there is no default value.
- [This check](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CErc20.sol#L130) in `CErc20.sol` is problematic for tokens that have multiple entry points (see https://github.com/d-xo/weird-erc20), as an admin can then take over all of the underlying tokens.
- Consider checking for the 0 address in [`setAccountantContract`](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CNote.sol#L20) to avoid errors.
- In `ProposalStore.go`, [this comment](https://github.com/Plex-Engineer/manifest-v2/blob/3aedf21400a470a50c9be6e38f1ca57f061eca46/contracts/ProposalStore.go#L14) refers to the wrong variable.
- In the function `doTransferIn` of the contract `CNote`, [this comment](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CNote.sol#L108) is wrong, as there is no underflow that can occur there.


# High: An attacker can render CNote’s doTransferOut unusable

URLs:

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CNote.sol#L148](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CNote.sol#L148)


## Impact
In `doTransferOut`, the underlying balance of the `CNote` has to be 0 after the transfer. While this works fine when the underlying balance of the `CNote` was 0 before the call (i.e., in the normal case), the function will always revert when the balance was greater than 0 before the transfer. An attacker can therefore block all withdrawals by sending a tiny amount of the underlying token to the CNote.

## Proof of Concept
User A holds multiple `CNote`s with an arbitrary underlying Token TKN. After some time, he wants to redeem his `CNote` for TKN and therefore calls `redeemFresh`. However, before he does that, an attacker transfers 1 TKN to the contract address of the `CNote`. In `doTransferOut`, `amount` is sent to the `CNote` and then transferred to user A. However, after the transfer, `token.balanceOf(address(this)) > 0`, meaning the transfer will always fail. 

## Recommended Mitigation Steps
Store the balance before the transfer and check that the difference is equal to the `amount`.


# High: getBorrowRate returns rate per year instead of per block

URLs:

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L118](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L118)

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CToken.sol#L209](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CToken.sol#L209)


## Impact
According to the documentation in `InterestRateModel`, `getBorrowRate` has to return the borrow rate per block and the function `borrowRatePerBlock` in `CToken` directly returns the value of `getBorrowRate`. However, the rate per year is returned for `NoteInterest`. Therefore, using `NoteInterest` as an interest model will result in completely wrong values. 

## Recommended Mitigation Steps
Return `baseRatePerBlock`.


# High: getSupplyRate returns rate per year instead of per block

URLs:

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L129](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L129)

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CToken.sol#L217](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/CToken.sol#L217)


## Impact
According to the documentation in `InterestRateModel`, `getSupplyRate` has to return the supply rate per block and the function `supplyRatePerBlock` in `CToken` directly returns the value of `getSupplyRate`. However, the rate per year is returned for `NoteInterest`. Therefore, using `NoteInterest` as an interest model will result in completely wrong values. 

## Recommended Mitigation Steps
Return `baseRatePerBlock`.


# Medium: baseRatePerBlock not updated when a new base rate is set

URLs:

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L169](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/NoteInterest.sol#L169)


## Impact
When an admin sets a new `baseRatePerYear` in `_setBaseRatePerYear`, the `baseRatePerBlock` is not updated. If the `deltaBlocks` has not passed yet, it will also not be updated when `getSupplyRate` is called, i.e. a wrong value will be returned there.

## Recommended Mitigation Steps
Update `baseRatePerBlock` to not return stale values.


# High: No authentication for SimplePriceOracle

URLs:

[https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/SimplePriceOracle.sol#L25](https://github.com/Plex-Engineer/lending-market-v2/blob/2646a7676b721db8a7754bf5503dcd712eab2f8a/contracts/SimplePriceOracle.sol#L25)


## Impact
Anyone can call `setUnderlyingPrice` on the `SimplePriceOracle` to set the oracle values. These are in turn used for the interest calculations, meaning anyone can manipulate this calculation via the Oracle.
Note that `SimplePriceOracle` in Compound is only intended as a test oracle and not for production (https://blog.openzeppelin.com/compound-tether-integration-audit/). If this also is true for this project, the issue would not be high, but from the deployment scripts, it looks to me as if the oracle is intended to be used in production.

## Recommended Mitigation Steps
Ensure that only trusted parties can update the prices or (even better) use a Chainlink Oracle.
