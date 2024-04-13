Dry Mint Koala

high

# OpenRelay does not take into account the L1 fees in L2 deployments

## Summary

## Vulnerability Detail
`OpenRelay.sol` calculates the fees for keeper with following calculations:
```solidity
    // Execute the automation vault counting the gas spent
    uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
    uint256 _gasSpent = _initialGas - gasleft();

    // Calculate the payment for the relayer
    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;

    // Send the payment to the relayer
    IAutomationVault.FeeData[] memory _feeData = new IAutomationVault.FeeData[](1);
    _feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
    _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);
```
Although this is sufficient for Mainnet, L2's gas payment calculations differ than mainnet and includes additional component (L1 fee) other than basefee. In fact this component is in most cases much bigger than the current calculation in `OpenRelay` that keepers can pay for gas more than 10x rewarded to them via `OpenRelay`. Because of this, `OpenRelay` in L2's are:
1. Useless for keepers that is aware of this, hence they won't call `exec`
2. Will lead to loss of funds for keepers that trusts to xKeeper's calculations because they will pay much more than what they are rewarded.
## Impact
Loss of funds for some keepers and broken core contract functionality for some Vault owners.
## Code Snippet
[OpenRelay.sol](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol/#L28-L39)
## Tool used

Manual Review

## Recommendation
Implement a mechanism for calculating L1 fees and take them into account. You can reference to related docs in [Optimism](https://docs.optimism.io/stack/transactions/fees), [Arbitrum](https://docs.arbitrum.io/arbos/l1-pricing), and in the chains that `OpenRelay` will be deployed in general. For example [Blast](https://docs.blast.io/building/guides/gas-fees) redirects some fees to the dapps, hence it also requires complete different deployment than both in current implementation, and also possible correct implementations for Optimism and Arbitrum.