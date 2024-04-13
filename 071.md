Colossal Rose Jaguar

medium

# `BASEFEE` opcode not supported on all EVM-compatible networks.

## Summary
Due to the `BASEFEE` opcode not being supported on some EVM-compatible network,  the `OpenRelay` contract will not be able to be used.

## Vulnerability Detail
When `OpenRelay.exec()` computes the payment to the keeper it uses the `BASEFEE` opcode to fetch the current block gas price.

```solidity
    uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
    uint256 _gasSpent = _initialGas - gasleft();


    // Calculate the payment for the relayer
    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
```

As described in the README the smart contracts will be deployed to "Any EVM-compatible network", however not all EVM-compatible network support the `BASEFEE` opcode, for example [Scroll](https://docs.scroll.io/en/developers/ethereum-and-scroll-differences/#evm-opcodes). On those networks all calls to `OpenRelay.exec()` will revert, thus making it impossible to be used.

## Impact
On those networks that do not support the `BASEFEE` opcode, like Scroll, `OpenRelay` won't be able to be used.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L21-L43

## Tool used
Manual Review

## Recommendation
Recommendations vary depending on the network being used. For deployment on the Scroll network, consider retrieving the gas values via the [RPC](https://docs.scroll.io/es/developers/guides/estimating-gas-and-tx-fees/#interfacing-with-values) rather than relying on the `BASEFEE` opcode
