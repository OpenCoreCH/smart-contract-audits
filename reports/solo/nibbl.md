# Scope
The audit focused on a Nibbl upgrade, introducing the following changes:
- New TWAV system with a minimum period between updates.
- New ERC1155Link contract, allowing to wrap / unwrap fractionalized tokens for different (configurable) tiers, represented as an ERC1155 token.
- Minor other changes to the Nibbl vault.
- General risks associated with these smart contract upgrades.

The code was delivered in the form of a GitHub repository: https://github.com/mundhrakeshav/nibbl-smartcontracts
Audited commit hash: `a65037a6e1b8bcc37c96c729533f3b7ef91185ba`

# Findings

During the audit, 1 issue with high severity, 2 issues with medium severity, and 4 issues with low severity were found. It is recommended that all issues with high and medium severity are addressed, whereas issues with low severity are up to the team's discretion.

## High Severity

### [H-01] `ERC1155Link` can be used to steal the tokens of other users
`ERC1155Link.wrap` always sets the token ID to 0 when minting:
```solidity
_mint(_to, 0, _amount, "0");
```
Similarly, it is set to 0 when burning the tokens in `unwrap`:
```solidity
_burn(msg.sender, 0, _amount);
```
For the tracking of the `totalSupply`, the token ID is respected. But because this is a counter that tracks the tokens of all users, an attacker can use this vulnerability to steal the ERC20 tokens of other users:

Let's say that there are two tiers of tokens, token ID 1 with a mint ratio of 10 and token ID 2 with a mint ratio of 50. Alice wraps 10 tokens with ID 2 and has to transfer 10 * 50 = 500 fractionalized tokens for that. The attacker Charlie wraps 10 tokens with ID 1 and has to transfer 10 * 10 = 100 fractionalized tokens for this operation. Charlie then calls `unwrap(10, 1, address(Charlie))`. This will call `totalSupply[1] -= 10` (which succeeds because the total supply of token 1 is 10), `_burn(address(charlie), 0, 10)` (which is also successful because Charlie owns 10 ERC1155 tokens with ID 0) and finally `linkErc20.transfer(_to, _amount * mintRatio[1]) = linkErc20.transfer(address(Charlie), 10 * 50)`. Because the mint ratio of token 1 is used (although Charlie never wrapped this token), he gets 500 fractionalized tokens instead of the 100 that he originally wrapped.

**Recommendation**: Use the provided token ID when minting / burning the ERC1155 tokens.

**Status**: Fixed

## Medium Severity

### [M-01] Curator for `ERC1155Link` not changed
With `updateCurator`, it is possible to change the curator of a Vault. However, the new curator will not be set for the `ERC1155Link` contract, which can be problematic in some cases (e.g., a compromised private key of the old curator, in which case the attacker could manipulate the `ERC1155Link` tiers).

**Recommendation**: Also change the curator for the `ERC1155Link` contract.

**Status**: Fixed

### [M-02] Multiple `ERC1155Link` contracts can be created for a vault
In `createERC1155Link`, it is not checked if an `ERC1155Link` for the vault was already deployed. A curator can therefore create a new one and overwrite the `nibblERC1155Link` variable. While this does not result in lost tokens (the user can still interact with the old contract to retrieve the ERC20 tokens), it can be very confusing, especially if this contract is displayed in the frontend in some way. Furthermore, this also means that the tier limits are not necessarily respected. When a user sees that a certain tier has a limit of 100 tokens, he cannot be sure that this will be enforced, because the curator could simply deploy a new link contract and recreate the tier there with a higher limit.

**Recommendation**: Consider only allowing creating the link contract when it does not exist already.

**Status**: Fixed

## Low Severity

### [L-01] Unnecessary comment in `Twav2.sol`
`Twav2.sol` contains the following commented out line:
```solidity
// uint256[50] private _gaps;
```
This is generally not problematic (because above this line, the variable is declared), but if the comment would be removed at some point, the whole vault would break.

**Recommendation**: Consider removing the comment to avoid errors in the future.

**Status**: Fixed

### [L-02] No more TWAV updates happening after timestamp rollover
In `NibblVault2`, the current timestamp is taken modulo 2^32 such that it rolls over:
```solidity
uint32 _blockTimestamp = uint32(block.timestamp % 2**32);
```
However, when this happens, no more TWAV updates will happen because the following statement will be `false`:
```solidity
if (_blockTimestamp >= lastBlockTimeStamp + period) {
```
While the consequences of this are severe (the vault completely stops working), it will only happen in February 2106.

**Recommendation:** In theory, the rollover could be incorporated into the check. However, because this will only be a problem in ~80 years and the contracts are upgradeable, leaving it as-is should also be fine.

**Status**: Acknowledged

### [L-03] Buyout result may already be decided two minutes before the end
Because of the new `period` parameter, when a TWAV update happens two minutes before the end of the buyout, the result is already decided (because no more TWAV updates are possible). However, it is still possible to buy / sell. This can lead to MEV opportunities (risk-free money when the buyout will succeed, but the current valuation is below the buyout price) and could in extreme cases even lead to situations where the valuation rises above the rejection value (because many want to cash in the risk-free money), which will be ignored because of the period.

However, this problem is not introduced thanks to the new TWAV system, it is only exacerbated. Even before, TWAV updates only happened for the first transaction within the block, so these MEV opportunities were also there in the last block before the buyout ended.

**Recommendation**: Avoiding this completely will be hard. A hybrid system (TWAV updates every two minutes, unless large movements happened) could help, but this would mean that the period is not always respected.

**Status**: Acknowledged

### [L-04] `ERC1155Link` ignores ERC20 return value
In the functions `wrap` and `unwrap`, the return value of `transfer` / `transferFrom` is not checked. While this is not problematic when the system is used in the current setup (in combination with `NibblVault`, which reverts instead of returning `false`), it is a violation of [EIP-20](https://eips.ethereum.org/EIPS/eip-20) which states that the return value must be checked. When the wrapper would be used with other tokens, this would introduce a major vulnerability.

**Recommendation**: Consider checking the transfer return value.

**Status**: Acknowledged

# Appendix
During the audit, the storage layout of `NibblVault` and `NibblVault2` was checked to ensure that the slots refer to the correct variables after the upgrade. This is the case, the two layouts are documented here:

**NibblVault**:
```json
{
  "storage": [
    {
      "astId": 11480,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_initialized",
      "offset": 0,
      "slot": "0",
      "type": "t_uint8"
    },
    {
      "astId": 11483,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_initializing",
      "offset": 1,
      "slot": "0",
      "type": "t_bool"
    },
    {
      "astId": 15424,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "__gap",
      "offset": 0,
      "slot": "1",
      "type": "t_array(t_uint256)50_storage"
    },
    {
      "astId": 48,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "maxExpArray",
      "offset": 0,
      "slot": "51",
      "type": "t_array(t_uint256)128_storage"
    },
    {
      "astId": 13261,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_balances",
      "offset": 0,
      "slot": "179",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 13267,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_allowances",
      "offset": 0,
      "slot": "180",
      "type": "t_mapping(t_address,t_mapping(t_address,t_uint256))"
    },
    {
      "astId": 13269,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_totalSupply",
      "offset": 0,
      "slot": "181",
      "type": "t_uint256"
    },
    {
      "astId": 13271,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_name",
      "offset": 0,
      "slot": "182",
      "type": "t_string_storage"
    },
    {
      "astId": 13273,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_symbol",
      "offset": 0,
      "slot": "183",
      "type": "t_string_storage"
    },
    {
      "astId": 13853,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "__gap",
      "offset": 0,
      "slot": "184",
      "type": "t_array(t_uint256)45_storage"
    },
    {
      "astId": 10909,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "twavObservationsIndex",
      "offset": 0,
      "slot": "229",
      "type": "t_uint8"
    },
    {
      "astId": 10914,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "lastBlockTimeStamp",
      "offset": 1,
      "slot": "229",
      "type": "t_uint32"
    },
    {
      "astId": 10920,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "twavObservations",
      "offset": 0,
      "slot": "230",
      "type": "t_array(t_struct(TwavObservation)10906_storage)6_storage"
    },
    {
      "astId": 10924,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "_gaps",
      "offset": 0,
      "slot": "242",
      "type": "t_array(t_uint256)50_storage"
    },
    {
      "astId": 11378,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "domainSeperator",
      "offset": 0,
      "slot": "292",
      "type": "t_bytes32"
    },
    {
      "astId": 3942,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "curveFee",
      "offset": 0,
      "slot": "293",
      "type": "t_uint256"
    },
    {
      "astId": 3945,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "secondaryReserveRatio",
      "offset": 0,
      "slot": "294",
      "type": "t_uint32"
    },
    {
      "astId": 3948,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "factory",
      "offset": 4,
      "slot": "294",
      "type": "t_address_payable"
    },
    {
      "astId": 3951,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "curator",
      "offset": 0,
      "slot": "295",
      "type": "t_address"
    },
    {
      "astId": 3954,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "assetAddress",
      "offset": 0,
      "slot": "296",
      "type": "t_address"
    },
    {
      "astId": 3957,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "assetID",
      "offset": 0,
      "slot": "297",
      "type": "t_uint256"
    },
    {
      "astId": 3960,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "bidder",
      "offset": 0,
      "slot": "298",
      "type": "t_address"
    },
    {
      "astId": 3963,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "initialTokenPrice",
      "offset": 0,
      "slot": "299",
      "type": "t_uint256"
    },
    {
      "astId": 3966,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "fictitiousPrimaryReserveBalance",
      "offset": 0,
      "slot": "300",
      "type": "t_uint256"
    },
    {
      "astId": 3969,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "buyoutRejectionValuation",
      "offset": 0,
      "slot": "301",
      "type": "t_uint256"
    },
    {
      "astId": 3972,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "buyoutValuationDeposit",
      "offset": 0,
      "slot": "302",
      "type": "t_uint256"
    },
    {
      "astId": 3975,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "initialTokenSupply",
      "offset": 0,
      "slot": "303",
      "type": "t_uint256"
    },
    {
      "astId": 3978,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "primaryReserveBalance",
      "offset": 0,
      "slot": "304",
      "type": "t_uint256"
    },
    {
      "astId": 3981,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "secondaryReserveBalance",
      "offset": 0,
      "slot": "305",
      "type": "t_uint256"
    },
    {
      "astId": 3984,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "feeAccruedCurator",
      "offset": 0,
      "slot": "306",
      "type": "t_uint256"
    },
    {
      "astId": 3987,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "buyoutEndTime",
      "offset": 0,
      "slot": "307",
      "type": "t_uint256"
    },
    {
      "astId": 3990,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "buyoutBid",
      "offset": 0,
      "slot": "308",
      "type": "t_uint256"
    },
    {
      "astId": 3993,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "curatorFee",
      "offset": 0,
      "slot": "309",
      "type": "t_uint256"
    },
    {
      "astId": 3996,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "totalUnsettledBids",
      "offset": 0,
      "slot": "310",
      "type": "t_uint256"
    },
    {
      "astId": 3999,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "minBuyoutTime",
      "offset": 0,
      "slot": "311",
      "type": "t_uint256"
    },
    {
      "astId": 4004,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "unsettledBids",
      "offset": 0,
      "slot": "312",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 4008,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "nonces",
      "offset": 0,
      "slot": "313",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 4015,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "status",
      "offset": 0,
      "slot": "314",
      "type": "t_enum(Status)4011"
    },
    {
      "astId": 4019,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "unlocked",
      "offset": 0,
      "slot": "315",
      "type": "t_uint256"
    },
    {
      "astId": 4021,
      "contract": "contracts/NibblVault.sol:NibblVault",
      "label": "imageUrl",
      "offset": 0,
      "slot": "316",
      "type": "t_string_storage"
    }
  ],
  "types": {
    "t_address": {
      "encoding": "inplace",
      "label": "address",
      "numberOfBytes": "20"
    },
    "t_address_payable": {
      "encoding": "inplace",
      "label": "address payable",
      "numberOfBytes": "20"
    },
    "t_array(t_struct(TwavObservation)10906_storage)6_storage": {
      "base": "t_struct(TwavObservation)10906_storage",
      "encoding": "inplace",
      "label": "struct Twav.TwavObservation[6]",
      "numberOfBytes": "384"
    },
    "t_array(t_uint256)128_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[128]",
      "numberOfBytes": "4096"
    },
    "t_array(t_uint256)45_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[45]",
      "numberOfBytes": "1440"
    },
    "t_array(t_uint256)50_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[50]",
      "numberOfBytes": "1600"
    },
    "t_bool": {
      "encoding": "inplace",
      "label": "bool",
      "numberOfBytes": "1"
    },
    "t_bytes32": {
      "encoding": "inplace",
      "label": "bytes32",
      "numberOfBytes": "32"
    },
    "t_enum(Status)4011": {
      "encoding": "inplace",
      "label": "enum NibblVault.Status",
      "numberOfBytes": "1"
    },
    "t_mapping(t_address,t_mapping(t_address,t_uint256))": {
      "encoding": "mapping",
      "key": "t_address",
      "label": "mapping(address => mapping(address => uint256))",
      "numberOfBytes": "32",
      "value": "t_mapping(t_address,t_uint256)"
    },
    "t_mapping(t_address,t_uint256)": {
      "encoding": "mapping",
      "key": "t_address",
      "label": "mapping(address => uint256)",
      "numberOfBytes": "32",
      "value": "t_uint256"
    },
    "t_string_storage": {
      "encoding": "bytes",
      "label": "string",
      "numberOfBytes": "32"
    },
    "t_struct(TwavObservation)10906_storage": {
      "encoding": "inplace",
      "label": "struct Twav.TwavObservation",
      "members": [
        {
          "astId": 10903,
          "contract": "contracts/NibblVault.sol:NibblVault",
          "label": "timestamp",
          "offset": 0,
          "slot": "0",
          "type": "t_uint32"
        },
        {
          "astId": 10905,
          "contract": "contracts/NibblVault.sol:NibblVault",
          "label": "cumulativeValuation",
          "offset": 0,
          "slot": "1",
          "type": "t_uint256"
        }
      ],
      "numberOfBytes": "64"
    },
    "t_uint256": {
      "encoding": "inplace",
      "label": "uint256",
      "numberOfBytes": "32"
    },
    "t_uint32": {
      "encoding": "inplace",
      "label": "uint32",
      "numberOfBytes": "4"
    },
    "t_uint8": {
      "encoding": "inplace",
      "label": "uint8",
      "numberOfBytes": "1"
    }
  }
}
```

**NibblVault2**:
```json
{
  "storage": [
    {
      "astId": 11480,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_initialized",
      "offset": 0,
      "slot": "0",
      "type": "t_uint8"
    },
    {
      "astId": 11483,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_initializing",
      "offset": 1,
      "slot": "0",
      "type": "t_bool"
    },
    {
      "astId": 15424,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "__gap",
      "offset": 0,
      "slot": "1",
      "type": "t_array(t_uint256)50_storage"
    },
    {
      "astId": 48,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "maxExpArray",
      "offset": 0,
      "slot": "51",
      "type": "t_array(t_uint256)128_storage"
    },
    {
      "astId": 13261,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_balances",
      "offset": 0,
      "slot": "179",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 13267,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_allowances",
      "offset": 0,
      "slot": "180",
      "type": "t_mapping(t_address,t_mapping(t_address,t_uint256))"
    },
    {
      "astId": 13269,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_totalSupply",
      "offset": 0,
      "slot": "181",
      "type": "t_uint256"
    },
    {
      "astId": 13271,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_name",
      "offset": 0,
      "slot": "182",
      "type": "t_string_storage"
    },
    {
      "astId": 13273,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_symbol",
      "offset": 0,
      "slot": "183",
      "type": "t_string_storage"
    },
    {
      "astId": 13853,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "__gap",
      "offset": 0,
      "slot": "184",
      "type": "t_array(t_uint256)45_storage"
    },
    {
      "astId": 11075,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "twavObservationsIndex",
      "offset": 0,
      "slot": "229",
      "type": "t_uint8"
    },
    {
      "astId": 11080,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "lastBlockTimeStamp",
      "offset": 1,
      "slot": "229",
      "type": "t_uint32"
    },
    {
      "astId": 11086,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "twavObservations",
      "offset": 0,
      "slot": "230",
      "type": "t_array(t_struct(TwavObservation)11069_storage)6_storage"
    },
    {
      "astId": 11090,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "_gaps",
      "offset": 0,
      "slot": "242",
      "type": "t_array(t_uint256)50_storage"
    },
    {
      "astId": 5767,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "storageFill",
      "offset": 0,
      "slot": "292",
      "type": "t_bytes32"
    },
    {
      "astId": 5806,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "curveFee",
      "offset": 0,
      "slot": "293",
      "type": "t_uint256"
    },
    {
      "astId": 5809,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "secondaryReserveRatio",
      "offset": 0,
      "slot": "294",
      "type": "t_uint32"
    },
    {
      "astId": 5812,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "factory",
      "offset": 4,
      "slot": "294",
      "type": "t_address_payable"
    },
    {
      "astId": 5815,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "curator",
      "offset": 0,
      "slot": "295",
      "type": "t_address"
    },
    {
      "astId": 5818,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "assetAddress",
      "offset": 0,
      "slot": "296",
      "type": "t_address"
    },
    {
      "astId": 5821,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "assetID",
      "offset": 0,
      "slot": "297",
      "type": "t_uint256"
    },
    {
      "astId": 5824,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "bidder",
      "offset": 0,
      "slot": "298",
      "type": "t_address"
    },
    {
      "astId": 5827,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "initialTokenPrice",
      "offset": 0,
      "slot": "299",
      "type": "t_uint256"
    },
    {
      "astId": 5830,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "fictitiousPrimaryReserveBalance",
      "offset": 0,
      "slot": "300",
      "type": "t_uint256"
    },
    {
      "astId": 5833,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "buyoutRejectionValuation",
      "offset": 0,
      "slot": "301",
      "type": "t_uint256"
    },
    {
      "astId": 5836,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "buyoutValuationDeposit",
      "offset": 0,
      "slot": "302",
      "type": "t_uint256"
    },
    {
      "astId": 5839,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "initialTokenSupply",
      "offset": 0,
      "slot": "303",
      "type": "t_uint256"
    },
    {
      "astId": 5842,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "primaryReserveBalance",
      "offset": 0,
      "slot": "304",
      "type": "t_uint256"
    },
    {
      "astId": 5845,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "secondaryReserveBalance",
      "offset": 0,
      "slot": "305",
      "type": "t_uint256"
    },
    {
      "astId": 5848,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "feeAccruedCurator",
      "offset": 0,
      "slot": "306",
      "type": "t_uint256"
    },
    {
      "astId": 5851,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "buyoutEndTime",
      "offset": 0,
      "slot": "307",
      "type": "t_uint256"
    },
    {
      "astId": 5854,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "buyoutBid",
      "offset": 0,
      "slot": "308",
      "type": "t_uint256"
    },
    {
      "astId": 5857,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "curatorFee",
      "offset": 0,
      "slot": "309",
      "type": "t_uint256"
    },
    {
      "astId": 5860,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "totalUnsettledBids",
      "offset": 0,
      "slot": "310",
      "type": "t_uint256"
    },
    {
      "astId": 5863,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "minBuyoutTime",
      "offset": 0,
      "slot": "311",
      "type": "t_uint256"
    },
    {
      "astId": 5868,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "unsettledBids",
      "offset": 0,
      "slot": "312",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 5872,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "nonces",
      "offset": 0,
      "slot": "313",
      "type": "t_mapping(t_address,t_uint256)"
    },
    {
      "astId": 5879,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "status",
      "offset": 0,
      "slot": "314",
      "type": "t_enum(Status)5875"
    },
    {
      "astId": 5882,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "unlocked",
      "offset": 0,
      "slot": "315",
      "type": "t_uint256"
    },
    {
      "astId": 5884,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "imageUrl",
      "offset": 0,
      "slot": "316",
      "type": "t_string_storage"
    },
    {
      "astId": 5886,
      "contract": "contracts/NibblVault2.sol:NibblVault2",
      "label": "nibblERC1155Link",
      "offset": 0,
      "slot": "317",
      "type": "t_address"
    }
  ],
  "types": {
    "t_address": {
      "encoding": "inplace",
      "label": "address",
      "numberOfBytes": "20"
    },
    "t_address_payable": {
      "encoding": "inplace",
      "label": "address payable",
      "numberOfBytes": "20"
    },
    "t_array(t_struct(TwavObservation)11069_storage)6_storage": {
      "base": "t_struct(TwavObservation)11069_storage",
      "encoding": "inplace",
      "label": "struct Twav2.TwavObservation[6]",
      "numberOfBytes": "384"
    },
    "t_array(t_uint256)128_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[128]",
      "numberOfBytes": "4096"
    },
    "t_array(t_uint256)45_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[45]",
      "numberOfBytes": "1440"
    },
    "t_array(t_uint256)50_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[50]",
      "numberOfBytes": "1600"
    },
    "t_bool": {
      "encoding": "inplace",
      "label": "bool",
      "numberOfBytes": "1"
    },
    "t_bytes32": {
      "encoding": "inplace",
      "label": "bytes32",
      "numberOfBytes": "32"
    },
    "t_enum(Status)5875": {
      "encoding": "inplace",
      "label": "enum NibblVault2.Status",
      "numberOfBytes": "1"
    },
    "t_mapping(t_address,t_mapping(t_address,t_uint256))": {
      "encoding": "mapping",
      "key": "t_address",
      "label": "mapping(address => mapping(address => uint256))",
      "numberOfBytes": "32",
      "value": "t_mapping(t_address,t_uint256)"
    },
    "t_mapping(t_address,t_uint256)": {
      "encoding": "mapping",
      "key": "t_address",
      "label": "mapping(address => uint256)",
      "numberOfBytes": "32",
      "value": "t_uint256"
    },
    "t_string_storage": {
      "encoding": "bytes",
      "label": "string",
      "numberOfBytes": "32"
    },
    "t_struct(TwavObservation)11069_storage": {
      "encoding": "inplace",
      "label": "struct Twav2.TwavObservation",
      "members": [
        {
          "astId": 11066,
          "contract": "contracts/NibblVault2.sol:NibblVault2",
          "label": "timestamp",
          "offset": 0,
          "slot": "0",
          "type": "t_uint32"
        },
        {
          "astId": 11068,
          "contract": "contracts/NibblVault2.sol:NibblVault2",
          "label": "cumulativeValuation",
          "offset": 0,
          "slot": "1",
          "type": "t_uint256"
        }
      ],
      "numberOfBytes": "64"
    },
    "t_uint256": {
      "encoding": "inplace",
      "label": "uint256",
      "numberOfBytes": "32"
    },
    "t_uint32": {
      "encoding": "inplace",
      "label": "uint32",
      "numberOfBytes": "4"
    },
    "t_uint8": {
      "encoding": "inplace",
      "label": "uint8",
      "numberOfBytes": "1"
    }
  }
}
```