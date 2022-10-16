# Foundation

# Minor Issues


- The size of the `__gap` differs per contract, some use 1000 uints (`NFTDropMarketCore`), other use 2000 (`FoundationTreasuryNode`). Consider using a unified size. Furthermore, using a smaller value such as 50 (what OpenZeppelin does) should be sufficient.
- In `getFeesAndRecipients`, the referral info is not included (https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L193)
- In `_getFees`, it is mentioned that "If the seller is unknown, assume it's being sold by the creator." ([https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L462](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L462)). However, this is actually not implemented. While this is not too bad as the seller is currently always known, it is a confusing comment.
- It is checked that the `buyReferrer` is not equal to `msg.sender` (https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L524). Of course, this check can easily be circumvented by providing a different address to always get a part of the fees.


# Medium: NFTDropCollection: _safeMint should be used

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/NFTDropCollection.sol#L182](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/NFTDropCollection.sol#L182)


## Impact
`NFTDropCollection` currently uses `_mint` instead of `_safeMint`. This is disencouraged, because `onERC721Received` is not called if the recipient is a smart contract. Therefore, when the recipient does not want to receive the NFT or has some custom business logic (e.g., internal accounting) on receive, this will be ignored.

## Recommended Mitigation Steps
Replace `_mint` with `_safeMint`. However, simply doing this would actually introduce a reentrancy vulnerability in `NFTDropMarketFixedPriceSale.mintFromFixedPriceSale` that would allow users to mint more than the `limitPerAccount`. Therefore, this function also needs a reentrancy guard in this case.


# Medium: postRevealBaseURIHash does not have to match revealed base URI

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/NFTDropCollection.sol#L200](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/NFTDropCollection.sol#L200)


## Impact
While the idea of pre-submitting a base URI hash is nice, it is currently not checked that the revealed base URI actually matches this hash. Therefore, the pre-submitted hash does not give the user any guarantees.

## Recommended Mitigation Steps
Only allow revealing when the hash of the baseURI matches the `postRevealBaseURIHash`. Do not allow changing this hash, unless it is currently 1.


# Medium: System will not work anymore after EIP-4758

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/collections/SequentialMintCollection.sol#L77](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/collections/SequentialMintCollection.sol#L77)


## Impact
After [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758), the `SELFDESTRUCT` op code will no longer be available. Therefore, the `_selfDestruct()` function will also no longer work and no more refunds will be possible (if there is any stuck ETH in the contract).

## Recommended Mitigation Steps
Consider replacing the `_selfDestruct()` function with a simple ETH transfer / rescue. In my opinion, this is the only valid use case the function currently has. Like that, it is ensured that your code is future-proof and will also work correctly after EIP-4758.


# Medium: Recreation of NFT collection with different parameters possible

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/254d384116d22266b12916f1aeb46b9ef0143f1f/contracts/NFTCollectionFactory.sol#L265](https://github.com/code-423n4/2022-08-foundation/blob/254d384116d22266b12916f1aeb46b9ef0143f1f/contracts/NFTCollectionFactory.sol#L265)


## Impact
Because an `NFTCollection` can be destroyed via `selfDestruct()`, it is possible for a malicious user to recreate a new collection at exactly the same address with different parameters (e.g., name or symbol) after it was destroyed. This can be easily misused.

## Proof Of Concept
Charlie creates an NFT collection and publishes the address. He front-runs the first mint from Bob, self-destructs the contracts and redeploys them with new parameters (e.g., a new name, symbol, or base URI). Because of that, Bob paid for an NFT that he did not want.

## Recommended Mitigation Steps
Either remove the `selfDestruct()` functionality or do not allow reusing the same nonce in `NFTCollectionFactory


# Medium: Treatment of ERC-2981 zero royalty amount

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L244](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L244)

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L302](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L302)


## Impact
If `royaltyInfo` returns 0 for the NFT contract, this is ignored. On the other hand, if 0 is returned for the override contract, the returned address gets 100% of the royalties. This should be unified to either always give 100% of the royalties in such a case or never do so. The latter option would be more sensible in my opinion, as the standard states: "Marketplaces that support this standard SHOULD NOT send a zero-value transaction if the royaltyAmount returned is 0. This would waste gas and serves no useful purpose in this EIP."
Meaning no transfer is expected when zero is returned.

## Recommended Mitigation Steps
Unify the business logic with regards to ERC-2981


# High: Creator fees may be burned

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L128](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L128)


## Impact
`royaltyInfo`, `getRoyalties`, or `getFeeRecipients` may return `address(0)` as the recipient address. While the value 0 is correctly handled for the royalties itself, it is not for the address. In such a case, the ETH amount will be sent to `address(0)`, i.e. it is burned and lost.

## Recommended Mitigation Steps
In your logic for determining the recipients, treat `address(0)` as if no recipient was returned such that the other priorities / methods take over.


# Medium: ERC-2981 violated

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L247](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L247)


## Impact
When `royaltyInfo` returns a recipient, he gets 100% of the royalties, i.e. 10% of the sale price. This violates ERC-2981, as the EIP clearly states that any protocol that integrates with it needs to transfer the amount that was returned: https://eips.ethereum.org/EIPS/eip-2981

Other protocols that assume EIP-2981 is followed may fail because of this (e.g., they assume that they always get 5% of a sale and use this assumption somewhere for calculations).

## Recommended Mitigation Steps
If the royaltyAmount is less than 10%, only return this amount and distribute the rest to the seller. If it is more, you would need to decide between reverting or allowing higher royalties.


# Medium: getFeeRecipients and owner potentially only called on override

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L335](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L335)

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L368](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L368)


## Impact
When there is an override contract (royalty lookup address), `getFeeRecipients` and `owner` are never called on the original NFT contract. This is in contrast to all other methods that are first called on the NFT contract itself and only on the override if the NFT contract does not return anything for them.

## Proof Of Concept
An NFT that implements `getFeeRecipients`, but also has an override contract is used by Bob in a transaction. Bob expects that the implementation calls `getFeeRecipients` on the NFT contract, as the NFT contract always takes precedence over the override. However, it is only calle don the override contract, meaning Bob's value is completely ignored.

## Recommended Mitigation Steps
Check `getFeeRecipients` and `owner` first on the NFT contract, even if there is a royalty lookup address.


# Medium: User may get all of the creator fees by specifying high number for himself

URLs:

[https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L486](https://github.com/code-423n4/2022-08-foundation/blob/7d6392498e8f3b8cdc22beb582188ffb3ed25790/contracts/mixins/shared/MarketFees.sol#L486)


## Impact
If one creator specified a share that is larger than `BASIS_POINTS`, the first creator gets all of the royalties. Depending on how these are set (which is not in the control of the project), this can be exploited by the first creator.

## Proof Of Concept
A collective of artists have implemented a process where everyone can set its own share of the fees to the value that he thinks is fair and these values are returned by `getRoyalties`. Bob, the first entry in the list of `_recipients` sets its value to a value that is slightly larger than `BASIS_POINTS` such that he gets all of the royalties.

## Recommended Mitigation Steps
There is no need for this check / logic, as the whole sum (`totalShares`) is used anyway to normalize the values.
