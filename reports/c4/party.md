# Party DAO

# Minor Issues


- In `PartyGovernance._getProposalStatus`, the constant `VETO_VALUE` is not used (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernance.sol#L1033), whereas it is used when performing the veto (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernance.sol#L634). While this is not a problem currently (the values are identical), it can be dangerous when this value is updated at one point, because updating `_getProposalStatus` might get forgotten. Consider using the constant consistently.
- The link for FractionalizeProposal (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/FractionalizeProposal.sol#L30) is wrong. It links to the source code of the vault, but the variable points to the factory. The correct link is https://github.com/fractional-company/contracts/blob/master/src/ERC721VaultFactory.sol
- After a `FractionalizeProposal` was executed succesfully and the tokens were distributed, it will not be possible to get the NFT back by calling `redeem` in practice. This function requires that the sender (the party in this case) owns all tokens, which would require that all users transfer them back again. However, transferring them to the party would be very risky when you do not if all other participants also do that (as the tokens are lost when one user does not transfer them) and some users may have sold them. This behavior may be desired (although it is a bit against the general philosophy were the tokens are protected IMO), but it should be documented that this is a destructive action.
- For Nouns, it is only possible to deploy the auction when the desired ID is for sale. However, it might be desired to create an auction in advance (e.g., for token ID 7 when the current ID is 5), such that there is more time to collect funds. For a system like Nouns this would work nicely, because it is (roughly) known in advance when an auction for a token will start.
- `CrowdfundNFT`, the implementation of a soulbound NFT, is a bit too simplistic in my opinion. One problem with soulbound NFTs is wallet recovery. In such situations, you want to be able to transfer the NFT to another wallet (but of course, this should not be possible without prior checks, otherwise they would not be soulbound). There are different standards like [EIP 4671](https://eips.ethereum.org/EIPS/eip-4671) that address soulbound NFTs (and the mentioned problems), consider implementing one of them.
- `FoundationMarketWrapper.auctionIdMatchesToken` returns `true` for auctionId 0 and `isFinalized` also returns `true`. Because of that, it is possible to create an auction with ID 0 where no one can bid because it is immediately finalized.
- `ZoraMarketWrapper.auctionIDMatchesToken` returns `true` for auctionId 0 and `nftContract = address(0)`, and `isFinalized` als returns `true`. Because of that, it is possible to create an auction with ID 0 and `address(0)` where no one can bid because it is immediately finalized.
- `BuyCrowdfund` cannot retrieve ETH, but certain contracts that are called buy it (e.g., NFT marketplaces) might return additional ETH when too much is paid. In such scenarios, the whole transaction would fail.


# Gas optimizations


- In `PartyGovernance.findVotingPowerSnapshotIndex`, a binary search is performed for the (common) case that `snaps[snaps.length - 1].timestamp <= timestamp`, i.e. when the timestamp is more recent than the latest checkpoint. Most other implementations (e.g, Compound) explicitly check this before performing a binary search, because it saves gas in expectation.
- In `Crowdfund._hashFixedGovernanceOpts`, `mload(opts)` is executed multiple times (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L331) instead of using the cached result.
- In the for loops, the loop iteration can be marked as `unchecked` because an overflow is not possible (as the iterator is bounded):
```
contracts/party/PartyGovernance.sol:306:        for (uint256 i=0; i < opts.hosts.length; ++i) {
contracts/distribution/TokenDistributor.sol:230:        for (uint256 i = 0; i < infos.length; ++i) {
contracts/distribution/TokenDistributor.sol:239:        for (uint256 i = 0; i < infos.length; ++i) {
contracts/crowdfund/Crowdfund.sol:180:        for (uint256 i = 0; i < contributors.length; ++i) {
contracts/crowdfund/Crowdfund.sol:242:        for (uint256 i = 0; i < numContributions; ++i) {
contracts/crowdfund/Crowdfund.sol:300:        for (uint256 i = 0; i < preciousTokens.length; ++i) {
contracts/crowdfund/Crowdfund.sol:348:            for (uint256 i = 0; i < numContributions; ++i) {
contracts/proposals/LibProposal.sol:14:        for (uint256 i = 0; i < preciousTokens.length; ++i) {
contracts/proposals/LibProposal.sol:32:        for (uint256 i = 0; i < preciousTokens.length; ++i) {
contracts/proposals/ArbitraryCallsProposal.sol:52:            for (uint256 i = 0; i < hadPreciouses.length; ++i) {
contracts/proposals/ArbitraryCallsProposal.sol:61:        for (uint256 i = 0; i < calls.length; ++i) {
contracts/proposals/ArbitraryCallsProposal.sol:78:            for (uint256 i = 0; i < hadPreciouses.length; ++i) {
contracts/proposals/ListOnOpenseaProposal.sol:291:            for (uint256 i = 0; i < fees.length; ++i) {
```



# Medium: CrowdFundNFT: EIP-721 violated

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundNFT.sol#L116](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundNFT.sol#L116)


## Impact & Proof Of Concept
For `tokenUri`, [EIP-721](https://eips.ethereum.org/EIPS/eip-721) states that it returns "A distinct Uniform Resource Identifier (URI) for a given asset. (...) URIs are defined in RFC 3986. The URI may point to a JSON file that conforms to the "ERC721 Metadata JSON Schema" ". However, the function returns directly a JSON object that conforms to the schema, it does not return a URI pointing to such a JSON file. This can cause major problems with third-party applications (e.g., marketplaces) which assume that EIP-721 is followed and can for instance cause scenarios where the tokens are not tradeable.

## Recommended Mitigation Steps
Consider an alternative design for `tokenUri`, e.g. returning an IPFS hash to the JSON file.


# Medium: AuctionCrowdfund: Fixed expiry can cause problems for marketplaces with dynamic expiration time

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L276](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L276)


## Impact & Proof Of Concept
The `expiry` timestamp of a crowdfund is fixed and set at deployment. However, for Nouns (NounsAuctionHouse.sol:128) and Foundation (NFTMarketReserveAuction.sol:246) the expiration of the auction can be extended and for Foundation, it only starts with the first bid. This can cause multiple problems:
- When an auction is longer than `expiry`, it is no longer possible to bid on it, even if the funds would be sufficient. This can be resolved by setting a very high `expiry`, but this causes other problems. In this case, all people have to wait until `expiry` to redeem their tokens if no bid was placed (because of https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L229).
- Another bidder can exploit this when he sees that his only rival bidder is the crowdfund: In such a scenario, he is incentivized to increase the bids by as little as possible and do that as late as possible to extend the auction as much as possible. When it reaches the `expiry` timestamp, the crowdfund can no longer bid (even if they would have the funds to do so), meaning that his rivals are eliminated. The crowdfund also cannot do anything against this (e.g., bidding a higher amount earlier), because the bid amount is restricted to the minimal increment.

## Recommended Mitigation Steps
The `expiry` should be (optionally) dynamic based on the expiration of the underlying auction. This would prevent the issues mentioned above.



# Medium: AuctionCrowdfund: Current design allows seller to always extract maximum bid amount

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L170](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L170)


## Impact
`CrowdfundNFT.bid` is callable by anyone. This design enables and incentivizes a seller to always sell his NFT for the maximum possible amount (wrt to the crowdfund), which hurts users of the Party protocol, as they pay potentially more than if they would not use the protocol.

## Proof Of Concept
There is a Crowdfund to bid on a bored ape with a `maximumBid` of 100 ETH and the crowdfund has accumulated 100 ETH. We assume that the platform has a minimum increment of 10%. The seller of the bored ape can now do the following:
- Get a flash loan for ~90.9090 ETH
- Bid ~90.9090 ETH for the ape
- Call `bid()` on the Crowdfund, the crowdfund will now bid 100 ETH
- Get back the ~90.9090 ETH bid and pay back the flash loan.

Like that, the seller can always sell his NFT risk-free and without locking up capital for `min(maximumBid, address(crowdfund).balance)`.

## Recommended Mitigation Steps
Only allow to call `bid` if the user has participated in the crowdsale (with a minimum threshold to avoid that a seller participates with 1 wei). While this does not eliminate this issue completely, it is no longer completely risk-free and the seller has to lock up his capital: If no user calls `bid()`, the sale will not succeed from the sellers perspective, although it might have if he had not placed this bid by himself.


# High: Possibility to burn all ETH in Crowdfund under some circumstances

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L147](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L147)


## Impact
If `opts.initialContributor` is set to `address(0)` (and `opts.initialDelegate` is not), there are two problems:
1.) If the crowdfund succeeds, the initial balance will be lost. It is still accredited to `address(0)`, but it is not retrievable. 
2.) If the crowdfund does not succeed, anyone can completely drain the contract by repeatedly calling `burn` with `address(0)`. This will always succeed because `CrowdfundNFT._burn` can be called multiple times for `address(0)`. Every call will cause the initial balance to be burned (transferred to `address(0)`).

Issue 1 is somewhat problematic, but issue 2 is very problematic, because all funds of a crowdfund are burned and an attacker can specifically set up such a deployment (and the user would not notice anything special, after all these are parameters that the protocol accepts).

## Proof Of Concept
This diff illustrates scenario 2, i.e. where a malicious deployer burns all contributions (1 ETH) of `contributor`. He loses 0.25ETH for the attack, but this could be reduced significantly (with more `burn(payable(address(0)))` calls:

```diff
--- a/sol-tests/crowdfund/BuyCrowdfund.t.sol
+++ b/sol-tests/crowdfund/BuyCrowdfund.t.sol
@@ -36,9 +36,9 @@ contract BuyCrowdfundTest is Test, TestUtils {
     string defaultSymbol = 'PBID';
     uint40 defaultDuration = 60 * 60;
     uint96 defaultMaxPrice = 10e18;
-    address payable defaultSplitRecipient = payable(0);
+    address payable defaultSplitRecipient = payable(address(this));
     uint16 defaultSplitBps = 0.1e4;
-    address defaultInitialDelegate;
+    address defaultInitialDelegate = address(this);
     IGateKeeper defaultGateKeeper;
     bytes12 defaultGateKeeperId;
     Crowdfund.FixedGovernanceOpts defaultGovernanceOpts;
@@ -78,7 +78,7 @@ contract BuyCrowdfundTest is Test, TestUtils {
                     maximumPrice: defaultMaxPrice,
                     splitRecipient: defaultSplitRecipient,
                     splitBps: defaultSplitBps,
-                    initialContributor: address(this),
+                    initialContributor: address(0),
                     initialDelegate: defaultInitialDelegate,
                     gateKeeper: defaultGateKeeper,
                     gateKeeperId: defaultGateKeeperId,
@@ -111,40 +111,26 @@ contract BuyCrowdfundTest is Test, TestUtils {
     function testHappyPath() public {
         uint256 tokenId = erc721Vault.mint();
         // Create a BuyCrowdfund instance.
-        BuyCrowdfund pb = _createCrowdfund(tokenId, 0);
+        BuyCrowdfund pb = _createCrowdfund(tokenId, 0.25e18);
         // Contribute and delegate.
         address payable contributor = _randomAddress();
         address delegate = _randomAddress();
         vm.deal(contributor, 1e18);
         vm.prank(contributor);
         pb.contribute{ value: contributor.balance }(delegate, "");
-        // Buy the token.
-        vm.expectEmit(false, false, false, true);
-        emit MockPartyFactoryCreateParty(
-            address(pb),
-            address(pb),
-            _createExpectedPartyOptions(0.5e18),
-            _toERC721Array(erc721Vault.token()),
-            _toUint256Array(tokenId)
-        );
-        Party party_ = pb.buy(
-            payable(address(erc721Vault)),
-            0.5e18,
-            abi.encodeCall(erc721Vault.claim, (tokenId)),
-            defaultGovernanceOpts
-        );
-        assertEq(address(party), address(party_));
-        // Burn contributor's NFT, mock minting governance tokens and returning
-        // unused contribution.
-        vm.expectEmit(false, false, false, true);
-        emit MockMint(
-            address(pb),
-            contributor,
-            0.5e18,
-            delegate
-        );
-        pb.burn(contributor);
-        assertEq(contributor.balance, 0.5e18);
+        vm.warp(block.timestamp + defaultDuration + 1);
+        // The auction was not won, we can now burn all ETH from contributor...
+        assertEq(address(pb).balance, 1.25e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 1e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.75e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.5e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.25e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0);
```

## Recommended Mitigation Steps
Do not allow an initial contribution when `opts.initialContributor` is not set.


# Medium: Bricked deployment when non-existing gateKeeperId is passed

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundFactory.sol#L45](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundFactory.sol#L45)

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundFactory.sol#L121](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/CrowdfundFactory.sol#L121)


## Impact & Proof Of Concept
A user can pass in a `gateKeeperId` which will be used (instead of creating a new one). However, the system does not check if this `gateKeeperId` actually exists. If this is not the case, the deployment is bricked and completely unusable. For `AllowListGateKeeper`, the merkle root will be 0, meaning that no addresses are allowed. For `TokenGateKeeper`, `gate.token` will be `address(0)`, causing every call to revert.

## Recommended Mitigation Steps
Check that the `gateKeeperId` exists.


# Medium: ArbitraryCallsProposal: onERC1155Received callable

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L194](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L194)


## Impact
In `ArbitraryCallsProposal`, any calls to `onERC721Received` are restricted to not trick contracts into thinking they received this token. However, calls to `onERC1155Received` can still be made. This is problematic because ERC1155 is backwards compatible with ERC721 (see Backwards Compatibility in https://eips.ethereum.org/EIPS/eip-1155) and there are hybrid ERC-1155/ERC-721 contracts (e.g., https://github.com/thesandboxgame/thesandbox-contracts/blob/master/src/Asset/ERC1155ERC721.sol)

## Proof Of Concept
Because these hybrid tokens exist, a smart contract that receives ERC1155 & ERC721 tokens might have a logic like that:
```
function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes memory data
    ) external returns (bytes4) {
		_handleIncomingNFT(operator, from, tokenId, data);
		return IERC721Receiver.onERC721Received.selector;
}

function onERC1155Received(address operator, address from, uint256 tokenId, uint256 value, bytes calldata data) external returns(bytes4) {
		if (isNFT[msg.sender] && value == 1) # isNFT is a mapping that stores ERC1155 tokens where balanceOf(tokenId) <= 1
			_handleIncomingNFT(operator, from, tokenId, data);
		...
		return IERC1155Receiver.onERC1155Received.selector;
}
```
In such a scenario, they could still be tricked into thinking that they received a NFT.

## Recommended Mitigation Steps
Also disallow calls to `onERC1155Received`.


# Medium: PartyGovernanceNFT: _mint instead of _safeMint used

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernanceNFT.sol#L132](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernanceNFT.sol#L132)


## Impact
`PartyGovernanceNFT.mint` currently uses `_mint` instead of `_safeMint`. This is disencouraged, because `onERC721Received` is not called if the recipient is a smart contract. Therefore, when the recipient does not want to receive the NFT or has some custom business logic (e.g., internal accounting) on receive, this will be ignored.

## Recommended Mitigation Steps
Replace `_mint` with `_safeMint`.


# Medium: Crowdfund: Truncated governanceOptsHash

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L333](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L333)


## Impact
The hash of the governance options is truncated to 16 bytes instead of using the full 32 bytes hash. Because of that, there are 2^128 different hash values and finding a collision takes 2^127 tries on average, i.e. a reduction by factor 2^128. While this nowadays still takes a long time, this may change in the future. The consequences of finding a collision would be severe. An attacker could provide a `hosts` array that contains an address that he controls and create the party with this array, giving him voting power (and the ability to remove the host status of all other hosts). Furthermore, he could set himself as the `feeRecipient` to get all fees.

## Proof Of Concept
We assume that in the future, an attacker with access to a supercomputer can calculate 2^110 hashes per second. The attacker fixes the first address of hosts to himself (and sets the other struct options to some values he desires, e.g. high fees and himself as the recipient). The second address of hosts is used by him to bruteforce the hashes, i.e. he simply tries different addresses there to get the desired hash.

This attack would take ~36 hours in expectation. If the full hash would be used, it would take over 10^36 years, i.e. 10^26 times the age of the universe.

## Recommended Mitigation Steps
Use the full 32 bytes hash.


# Medium: Token can be sold under value when GLOBAL_OS_ZORA_AUCTION_DURATION is 0

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnOpenseaProposal.sol#L159](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnOpenseaProposal.sol#L159)


## Impact
When `LibGlobals.GLOBAL_OS_ZORA_AUCTION_TIMEOUT > 0`, but `LibGlobals.GLOBAL_OS_ZORA_AUCTION_DURATION = 0`, an auction on Zora will still be created. This can lead to undesired situations:
When an auction has duration 0 on Zora, it is only possible to place one bid (see https://github.com/smartdev0530/zora-contracts/blob/main/AuctionHouse.sol#L165), not even a second one in the same block can be placed.

## Proof Of Concept
A malicious party member that owns a supermajority wants to get an NFT out cheaply. While he cannot do that with `ListOnZoraProposal` (because of the minimum duration), he can do so with `ListOnOpenseaProposal` when `LibGlobals.GLOBAL_OS_ZORA_AUCTION_TIMEOUT > 0` and `LibGlobals.GLOBAL_OS_ZORA_AUCTION_DURATION = 0`. To do so, he creates a `ListOnOpenseaProposal` with a very low list price. Immediately afterwards, he bids this list price (which is the only allowed bid by Zora) to get the NFT almost for free.

## Recommended Mitigation Steps
Do not create a zora auction when `LibGlobals.GLOBAL_OS_ZORA_AUCTION_DURATION == 0`.


# Medium: Lost ETH when NFT was acquired for free

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/BuyCrowdfundBase.sol#L122](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/BuyCrowdfundBase.sol#L122)

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L344](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L344)

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L236](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/AuctionCrowdfund.sol#L236)


## Impact
When a NFT was acquired for free in `BuyCrowdfundBase`, `settledPrice` is set to the contributions such that everyone who contributed wins. However, this has also the side effect that all of the contributed ETH is lost. It was not used for buying the NFT, but it also is not returned to the users when calling `_burn` (because all of their contributions are smaller than `_getFinalPrice()`, as `_getFinalPrice()` returns the total of all contributions).

Note that the same problem also exists in `AuctionCrowdfund`.

## Proof Of Concept
A party decides to buy a certain NFT and 100 members contribute 1 ETH each. Because the owner of this NFT is also a member of the party and knows all these persons personally, he decides to gift the NFT to them (by providing a function that is only callable by the party). While the party gets the NFT, the 100 ETH that they contributed is lost.

## Recommended Mitigation Steps
In my opinion, there are two possible strategies:
1.) Introduce a special flag for such situations, allow the users to redeem all of their ETH in `_burn`, while still getting the corresponding voting power.
2.) In such situations, transfer the whole ETH balance to the party where it can be distributed (preferrably in my opinion)


# High: PartyGovernance: Can vote multiple times by transferring NFT in same block as proposal

URLs:

https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernance.sol#L594


## Impact
`PartyGovernanceNFT` uses the voting power at the time of proposal when calling `accept`. The problem with that is that a user can vote, transfer the NFT (and the voting power) to a different wallet, and then vote from this second wallet again during the same block that the proposal was created. 
This can also be repeated multiple times to get an arbitrarily high voting power and pass every proposal unanimously. 

The consequences of this are very severe. Any user (no matter how small his voting power is) can propose and pass arbitrary proposals animously and therefore steal all assets (including the precious tokens) out of the party.

## Proof Of Concept
This diff shows how a user with a voting power of 50/100 gets a voting power of 100/100 by transferring the NFT to a second wallet that he owns and voting from that one:
```diff
--- a/sol-tests/party/PartyGovernanceUnit.t.sol
+++ b/sol-tests/party/PartyGovernanceUnit.t.sol
@@ -762,6 +762,7 @@ contract PartyGovernanceUnitTest is Test, TestUtils {
         TestablePartyGovernance gov =
             _createGovernance(100e18, preciousTokens, preciousTokenIds);
         address undelegatedVoter = _randomAddress();
+        address recipient = _randomAddress();
         // undelegatedVoter has 50/100 intrinsic VP (delegated to no one/self)
         gov.rawAdjustVotingPower(undelegatedVoter, 50e18, address(0));
 
@@ -772,38 +773,13 @@ contract PartyGovernanceUnitTest is Test, TestUtils {
         // Undelegated voter submits proposal.
         vm.prank(undelegatedVoter);
         assertEq(gov.propose(proposal, 0), proposalId);
-
-        // Try to execute proposal (fail).
-        vm.expectRevert(abi.encodeWithSelector(
-            PartyGovernance.BadProposalStatusError.selector,
-            PartyGovernance.ProposalStatus.Voting
-        ));
-        vm.prank(undelegatedVoter);
-        gov.execute(
-            proposalId,
-            proposal,
-            preciousTokens,
-            preciousTokenIds,
-            "",
-            ""
-        );
-
-        // Skip past execution delay.
-        skip(defaultGovernanceOpts.executionDelay);
-        // Try again (fail).
-        vm.expectRevert(abi.encodeWithSelector(
-            PartyGovernance.BadProposalStatusError.selector,
-            PartyGovernance.ProposalStatus.Voting
-        ));
-        vm.prank(undelegatedVoter);
-        gov.execute(
-            proposalId,
-            proposal,
-            preciousTokens,
-            preciousTokenIds,
-            "",
-            ""
-        );
+        (, PartyGovernance.ProposalStateValues memory valuesPrev) = gov.getProposalStateInfo(proposalId);
+        assertEq(valuesPrev.votes, 50e18);
+        gov.transferVotingPower(undelegatedVoter, recipient, 50e18); //Simulate NFT transfer
+        vm.prank(recipient);
+        gov.accept(proposalId, 0);
+        (, PartyGovernance.ProposalStateValues memory valuesAfter) = gov.getProposalStateInfo(proposalId);
+        assertEq(valuesAfter.votes, 100e18);
     }
```

## Recommended Mitigation Steps
You should query the voting power at `values.proposedTime - 1`. This value is already finalized when the proposal is created and therefore cannot be manipulated by repeatedly transferring the voting power to different wallets.


# Medium: Possible that unanimous votes is unachievable

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L370](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L370)

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernance.sol#L1066](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/party/PartyGovernance.sol#L1066)


## Impact
Currently, the `votingPower` calculation rounds down for every user except the `splitRecipient`. This can cause situations where 99.99% of votes (i.e., an unanimous vote) are not achieved, even if all vote in favor of a proposal. This can be very bad for a party, as certain proposals (transferring precious tokens out) require an unamimous vote and are therefore not executable.

## Proof Of Concept
Let's say for the sake of simplicity that 100 persons contribute 2 wei and `splitBps` is 10 (1%). `votingPower` for all users that contributed will be 1 and 2 for the `splitRecipient`, meaning the maximum achievable vote percentage is 102 / 200 = 51%.

Of course, this is an extreme example, but it shows that the current calculation can introduce siginificant rounding errors that impact the functionality of the protocol.

## Recommended Mitigation Steps
Instead of requiring more than 99.99% of the votes, ensure that the individual votingPower sum to the total contribution. For instance, one user (e.g., the last one to claim) could receive all the remaining votingPower, which would require a running sum of the already claimed votingPower.



# Medium: ArbitraryCallsProposal may fail, although enough ETH would be available

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L72](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L72)


## Impact
In `ArbitraryCallsProposal`, `ethAvailable` is reduced by `calls[i].value`, i.e. the amount that is sent with the call. However, there are certain contracts that sent back ETH (e.g., reimbursing a user when he paid too much). In such a situation, the call would revert because of too little ETH, although there is enough.

## Proof Of Concept
`ArbitraryCallsProposal` is used for a proposal that buys five NFTs on a marketplace like Rarible. The party wants to buy at most 5 ETH for all NFTs. Because the final price of them is not fixed at the time of the proposal, the proposer decides to attach 1.05 ETH to each call, such that there is some leeway. Additional ETH will be reimbursed by Rarible, so this proposal should nicely work for the intended purpose (it reverts when the total price of all NFTs is higher than 5 ETH and executs succesfully, otherwise). When the proposal is executed, the total cost for all NFTs is 4.95 ETH and the contract. However, it does not execute succesfully because 1.05 ETH is deducted after every call and `ethAvailable -= calls[i].value` therefore underflows after the fifth call.

## Recommended Mitigation Steps
Subtract the actual balance differences (before and after each call), not the amount that was sent.


# Medium: getMinimumBid can return invalid value for Zora

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/market-wrapper/ZoraMarketWrapper.sol#L80](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/market-wrapper/ZoraMarketWrapper.sol#L80)


## Impact & Proof Of Concept
`getMinimumBid` simply returns the reserve price (when there are no bidders) or the minimum increment. However, when the `tokenContract` is equal to the zora protocol (which is possible, as this is not disallowed by the Party protocol), there is an additional check in Zora's [AuctionHouse](https://github.com/smartdev0530/zora-contracts/blob/main/AuctionHouse.sol#L180):
```solidity
				// For Zora Protocol, ensure that the bid is valid for the current bidShare configuration
        if(auctions[auctionId].tokenContract == zora) {
            require(
                IMarket(IMediaExtended(zora).marketContract()).isValidBid(
                    auctions[auctionId].tokenId,
                    amount
                ),
                "Bid invalid for share splitting"
            );
        }
```
Looking at Zora's [Market](https://github.com/smartdev0530/zora-contracts/blob/main/Market.sol#L91), we can see that this checks if the amount is perfectly splittable:
```solidity
	/**
     * @notice Validates that the bid is valid by ensuring that the bid amount can be split perfectly into all the bid shares.
     *  We do this by comparing the sum of the individual share values with the amount and ensuring they are equal. Because
     *  the splitShare function uses integer division, any inconsistencies with the original and split sums would be due to
     *  a bid splitting that does not perfectly divide the bid amount.
     */
    function isValidBid(uint256 tokenId, uint256 bidAmount)
        public
        view
        override
        returns (bool)
    {
        BidShares memory bidShares = bidSharesForToken(tokenId);
        require(
            isValidBidShares(bidShares),
            "Market: Invalid bid shares for token"
        );
        return
            bidAmount != 0 &&
            (bidAmount ==
                splitShare(bidShares.creator, bidAmount)
                    .add(splitShare(bidShares.prevOwner, bidAmount))
                    .add(splitShare(bidShares.owner, bidAmount)));
    }
```

It is therefore possible that the amount returned by `getMinimumBid` does not pass this check, which would mean that it is impossible to bid on this token (as Party protocol always uses the minimum amount and there is no way to pass an amount manually).

## Recommended Mitigation Steps
Incorporate this into the `getMinimumBid` calculation. If `tokenContract` is equal to the Zora protocol, the amount returned has to be larger than the minimum increment AND perfectly splittable.


# High: _settleZoraAuction returns true when it reverts with "Auction hasn’t completed”

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnZoraProposal.sol#L199](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnZoraProposal.sol#L199)


## Impact
Zora's [AuctionHouse.endAuction](https://github.com/ourzora/auction-house/blob/main/contracts/AuctionHouse.sol#L257) can revert with "Auction hasn't completed" in case an auction has not completed yet:
```
require(
            block.timestamp >=
            auctions[auctionId].firstBidTime.add(auctions[auctionId].duration),
            "Auction hasn't completed"
        );
```
However, because this `errHash` is not handled in `_settleZoraAuction`, the function will return `true` in such cases (indicating that the NFT was succesfully sold). This has multiple negative implications:
1.) `_executeListOnZora` will not revert in such a scenario (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnZoraProposal.sol#L123) and return an empty string, indicating that there is no next step. Therefore, no further calls for this proposal are allowed, although it is still in progress.
2.) When the function is called within `_executeListOnOpensea` (https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ListOnOpenseaProposal.sol#L182), the OpenSea proposal will be stopped, even if the auction did not succeed (or will not succeed in the future). This can potentially block progress on all OpenSea proposals.

## Proof Of Concept
Implication 1 can be exploited to transfer a NFT out with an `ArbitraryCallsProposal`. Normally, the following attack would not be possible because only one proposal can be in progress at any given moment. But because of the issue mentioned above, the system thinks that the Zora auction has succeeded, when it is still in progress (and may still fail). Let's assume that the in progress auction will ultimately fail, i.e. the following code in Zora's AuctionHouse (which transfers the token back to the party) will be triggered:
```
				_handleOutgoingBid(auctions[auctionId].bidder, auctions[auctionId].amount, auctions[auctionId].auctionCurrency);
                _cancelAuction(auctionId);
                return;
```

### Exploitation with an ArbitraryCallsProposal
As a security measure, it is not possible to transfer precious tokens out with an `ArbitraryCallsProposal`. However, this measure works by checking first if the token was owned previously and only reverts if it was owned previously and is not owned after the execution. If we are in the situation described above, a party member can circumvent this measure with the following `ArbitraryCallsProposal`:
1.) Call `endAuction` on the auction. As mentioned above, we assume that this fails (for instance, because the bidder rejects the NFT, we could also specifically set up such a scenario as an attacker).
2.) Transfer the precious token to an address that the party member owns.

Note that is `ArbitraryCallsProposal` succeeds because the precious token was previously not owned by the party (the Zora auction house was still the owner). A party member that has 51% of the votes can therefore transfer a precious token out with the described attack (this would normally require an unanimous vote).

## Recommended Mitigation Steps
Handle a reversion with reason "Auction hasn't completed" correctly. In such a situation, `_settleZoraAuction` should also revert.


# High: ArbitraryCallsProposal can be used in combination with FractionalizeProposal to get precious token out without an unanimous vote

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L142](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/proposals/ArbitraryCallsProposal.sol#L142)


## Impact
`ArbitraryCallsProposal._isCallAllowed` does not disallow ERC20 transfers. This can be exploited in combination with a `FractionalizeProposal` to get a precious token without an unanimous vote, which should not be possible.

## Proof Of Concept
There are two different attack possibilities:

### Transferring Fractionalize tokens out
- A user with 51% voting rights creates a `FractionalizeProposal` that succeeds, the party gets all of the ERC20 Fractionalize tokens.
- Afterwards, he creates an `ArbitraryCallsProposal` to transfer all ERC20 Fractionalize to a wallet that he owns.
- Now, he can call `redeem` on Fractionalize (https://github.com/fractional-company/contracts/blob/master/src/ERC721TokenVault.sol#L371) to get the NFT back.

### ArbitraryCallsProposal to call redeem
- A user with 51% voting rights creates a `FractionalizeProposal` that succeeds, the party gets all of the ERC20 Fractionalize tokens.
- Afterwards, he creates an `ArbitraryCallsProposal` that does two things:
	- `redeem` is called on Fractionalize (https://github.com/fractional-company/contracts/blob/master/src/ERC721TokenVault.sol#L371), the NFT is transferred back to the party
	- The user transfers the NFT to his wallet.

Note that this attack succeeds because the party was not in possession of the NFT before the `ArbitraryCallsProposal` was executed, meaning that it will not be checked if it is in possesion after the execution.

## Recommended Mitigation Steps
For attack 1, disallow ERC20 transfers (this should be done via the `TokenDistributor`). For attack 2, disallow calls to the Fractionalize vault via an `ArbitraryCallsProposal`.


# High: TokenDistributor: ERC777 tokensToSend hook can be exploited to drain contract

URLs:

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/distribution/TokenDistributor.sol#L131](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/distribution/TokenDistributor.sol#L131)

[https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/distribution/TokenDistributor.sol#L386](https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/distribution/TokenDistributor.sol#L386)


## Impact
`TokenDistributor.createERC20Distribution` can be used to create token distributions for ERC777 tokens (which are backwards-compatible with ERC20). However, this introduces a reentrancy vulnerability which allows a party to get the tokens of another party. The problem is the `tokensToSend` hook which is executed BEFORE balance updates happens (see https://eips.ethereum.org/EIPS/eip-777). When this hook is executed, `token.balanceOf(address(this))` therefore still returns the old value, but `_storedBalances[balanceID]` was already decreased.

## Proof Of Concept
Party A and Party B have a balance of 1,000,000 tokens (of some arbitrary ERC777 token) in the distributor. Let's say for the sake of simplicity that both parties only have one user (user A in party A, user B in party B). User A (or rather his smart contract) performs the following attack:
- He calls `claim`, which transfers 1,000,000 tokens to his contract address. In `_transfer`, `_storedBalances[balanceId]` is decreased by 1,000,000 and therefore now has a value of 1,000,000.
- In the `tokensToSend` hook, he initiates another distribution for his party A by calling `PartyGovernance.distribute` which calls `TokenDistributor.createERC20Distribution` (we assume for the sake of simplicity that the party does not have more of these tokens, so the call transfers 0 tokens to the distributor). `TokenDistributor.createERC20Distribution` passes `token.balanceOf(address(this))` to `_createDistribution`. Note that this is still 2,000,000 because we are in the `tokensToSend` hook.
- The supply of this distribution is calculated as `(args.currentTokenBalance - _storedBalances[balanceId]) = 2,000,000 - 1,000,000 = 1,000,000`.
- When the `tokensToSend` hook is exited (and the first transfer has finished), he can retrieve the tokens of the second distribution (that was created in the hook) to steal the 1,000,000 tokens of party B.

## Recommended Mitigation Steps
Do not allow reentrancy in these functions.
