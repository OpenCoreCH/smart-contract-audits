# Axelar

# Medium: Change of operators possible from old operators

URLs:

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/AxelarGateway.sol#L268](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/AxelarGateway.sol#L268)

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/AxelarGateway.sol#L311](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/AxelarGateway.sol#L311)


## Impact
According to the specifications, only the current operators should be able to transfer operatorship. However, there is one way to circumvent this. Because currentOperators is not updated in the loop, when multiple `transferOperatorship` commands are submitted in the same `execute` call, all will succeed. After the first one, the operators that signed these commands are no longer the current operators, but the call will still succeed.

This also means that one set of operators could submit so many `transferOperatorship` commands in one `execute` call that `OLD_KEY_RETENTION` is reached for all other ones, meaning they would control complete set of currently valid operators.

## Recommended Mitigation Steps
Set `currentOperators` to `false` when the operators were changed.


# Medium: System will not work anymore after EIP-4758

URLs:

[https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/DepositReceiver.sol#L25](https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/DepositReceiver.sol#L25)


## Impact
After [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758), the `SELFDESTRUCT` op code will no longer be available. According to the EIP, "The only use that breaks is where a contract is re-created at the same address using CREATE2 (after a SELFDESTRUCT).". Axelar is exactly such an application, the current deposit system will no longer work.

## Recommended Mitigation Steps
To avoid that Axelar simply stops working one day, the architecture should be changed. Instead of generating addresses for every user, the user could directly interact with the deposit service and the deposit service would need to keep track of funds and provide refunds directly.


# Medium: Deprecated transfer in various places used

URLs:

[https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L23](https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L23)

[https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L51](https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L51)

[https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L71](https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L71)

[https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L86](https://github.com/code-423n4/2022-07-axelar/blob/a46fa61e73dd0f3469c0263bc6818e682d62fb5f/contracts/deposit-service/ReceiverImplementation.sol#L86)

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L128](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L128)

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L144](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L144)

[https://github.com/code-423n4/2022-07-axelar/blob/a1205d2ba78e0db583d136f8563e8097860a110f/xc20/contracts/XC20Wrapper.sol#L63](https://github.com/code-423n4/2022-07-axelar/blob/a1205d2ba78e0db583d136f8563e8097860a110f/xc20/contracts/XC20Wrapper.sol#L63)


## Impact
The system uses `transfer` which only has a 2300 gas stipend in various places for transferring ETH. Depending on the logic of the retriever (e.g., a smart contract that performs some storage read and writes on receive, for instance a multi-sig), this may not be sufficient and the transfer can revert. See also https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/ for a discussion of this issue.

## Recommended Mitigation Steps
Use `call` instead, e.g.:

```solidity
(bool success, ) = payable(payAddress).call{amount}("");
require(success, "Transfer failed.");
```


# Medium: Wrong gasFeeAmount emitted for tokens with fees

URLs:

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L21](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L21)

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L46](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L46)

[https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L105](https://github.com/code-423n4/2022-07-axelar/blob/3373c48a71c07cfce856b53afc02ef4fc2357f8c/contracts/gas-service/AxelarGasService.sol#L105)


## Impact
Some ERC20 tokens charge a fee on transfer (e.g. STA or PAXG) and some may start doing so in the future (e.g. USDT or USDC). When such tokens are used for paying fees, the system will still emit the supplied `gasFeeAmount`, even if less was actually transferred. This will cause relayers to relay the transaction, even if too little gas was paid.

## Recommended Mitigation Steps
Check how much the balance actually increased and emit that.


# Medium: removeWrapping can be called when there are still wrapped tokens

URLs:

[https://github.com/code-423n4/2022-07-axelar/blob/a1205d2ba78e0db583d136f8563e8097860a110f/xc20/contracts/XC20Wrapper.sol#L66](https://github.com/code-423n4/2022-07-axelar/blob/a1205d2ba78e0db583d136f8563e8097860a110f/xc20/contracts/XC20Wrapper.sol#L66)


## Impact
An owner can call `removeWrapping`, even if there are still circulating wrapped tokens. This will cause the unwrapping of those tokens to fail, as `unwrapped[wrappedToken]` will be `addres(0)`.

## Recommended Mitigation Steps
Track how many wrapped tokens are in circulation, only allow the removal of a wrapped tokens when there are 0 to ensure for users that they will always be able to unwrap.
