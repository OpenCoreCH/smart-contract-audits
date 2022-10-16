# Nibbl

# Minor Issues


- `block.timestamp % 2**32` (https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L445) will reset to 0 on Feb 07, 2106. This results in completely wrong values for the TWAV calculation (as it determines the seconds passed as `2**32` in such a case).
- In contrast to all other values that are relevant for a buyout, `buyoutValuationDeposit` is not deleted in `_rejectBuyout()` (https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L424)
- In multiple places, there are no zero checks when an address is set. Consider adding a check there to avoid mistakes:
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L24
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L25
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L26
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L100
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L124
	- https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L159
- There is no way for the role admin to remove a role proposal (https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/Utilities/AccessControlMechanism.sol#L41)


# Medium: No fees deducted for secondary curve when buying

URLs:

- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L319](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L319)


## Impact
When a buy can be filled only partially by the secondary curve and the rest is filled by the primary curve, no admin and curator fee is deducted for the part that is filled by the secondary curve (https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L319). Therefore, a user could avoid buying fees almost completely for certain orders (that increase the secondary curve just above the maximum value).

## Recommended Mitigation Steps
Also deduct a fee in the linked branch.


# Medium: Buyout cannot be rejected when paused

URLs:

- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L300](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L300)
- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L362](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L362)
- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L464](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L464)
- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L495](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVault.sol#L495)


## Impact
While `buy()` and `sell()` are only callable when the system is not paused, `redeem()` and `withdrawERC721()` are also callable when it is not. This means that the `BUYOUT_DURATION` is ignored in such cases and it is possible that users are not able to reject certain buyouts.

## Proof of Concept
A user initiates a buyout via `initiateBuyout()`. Just afterwards, the system is stopped. The token holders now cannot buy new tokens to increase the value. However, after two days, the `bidder` can still withdraw the NFT, i.e. there was no way for the users to reject this buyout.

## Recommended Mitigation Steps
It should be possible to reset the `buyoutEndTime` (to the current `block.timestamp`) when the system is paused such that the token holders always have the possibility to reject a buyout.



# Medium: Basket address for a deployed basket changes when the implementation changes

URLs:

- [https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L88](https://github.com/code-423n4/2022-06-nibbl/blob/71b639f977c0351c9928dd3b78eaa4bebb738bc1/contracts/NibblVaultFactory.sol#L88)


## Impact
When the `basketImplementation` is changed after a basket is deployed, `getBasketAddress` will return a different address (because of the changed `creationCode` and address that is used to deterministically compute the address).
This can have major consequences. When this function is used in the frontend / by interacting applications, they will suddenly point to a new, non-existing address and the user can think that the NFTs are lost.

## Recommended Mitigation Steps
Consider caching the basket addresses after deployment
