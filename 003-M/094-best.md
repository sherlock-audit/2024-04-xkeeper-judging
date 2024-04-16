Fierce Pear Troll

medium

# Hard coded value of `GAS_BONUS` in `OpenRelay.sol` makes the contract less adaptable to changes in the Ethereum protocol that might affect gas costs

## Summary
The hardcoded value of `GAS_BONUS` in the [`OpenRelay.sol`](https://github.com/defi-wonderland/xkeeper-core/blob/dev/solidity/contracts/relays/OpenRelay.sol) contract can potentially become an issue, especially if the Ethereum network undergoes changes that affect opcode gas costs, such as Ethereum Improvement Proposals (EIPs) that modify gas pricing. Given the dynamic nature of Ethereum's gas market and the potential for protocol updates, relying on a hardcoded value might not be flexible or responsive enough to adapt to changes efficiently. 

## Vulnerability Detail
The `OpenRelay` contract incentivizes [120% of the gas costs, paid out in ETH instantaneously](https://docs.xkeeper.network/content/how-to/open_relay.html) to the executor of a job. It does this by calculating the payment as :

[`OpenRelay.sol::exec`](https://github.com/defi-wonderland/xkeeper-core/blob/d7572f7af607dc69377636bb0e186d45600388b1/solidity/contracts/relays/OpenRelay.sol#L21) : 
```solidity
    uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
    uint256 _gasSpent = _initialGas - gasleft();

    // Calculate the payment for the relayer
    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
```
Here the `_gasSpent` variable stores the gas spent to execute the specified jobs in the automation vault, and the `GAS_BONUS` accounts for the gas paid by the executor for the function call(calling `OpenRelay.sol::exec`) and the second `_automationVault.exec`(to pay the fees for the jobs). 

By doing this, the net payment made to the executor currently is 120% of the total gas they spent on the transaction.

However, If the gas costs for operations change due to protocol updates, the hardcoded `GAS_BONUS` might no longer reflect the actual costs incurred by a caller, potentially leading to under-compensation or over-compensation.

## Impact
Increased incentive% may be exploited to drain the automation vault, whereas decreased incentive% might disincentivize users from completing jobs.
## Code Snippet

## Tool used

Manual Review

## Recommendation
Make `GAS_BONUS` a mutable variable