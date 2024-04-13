High Honey Scorpion

medium

# Bots using the OpenRelay are highly susceptible to griefing attacks

## Summary

Keepers that operate using the OpenRelay can be tricked into executing a malicious job that could eventually fail while executed on-chain, causing them to waste gas costs without getting any payment.

## Vulnerability Detail

The OpenRelay has a simple design in which the relay calculates the amount of gas spent while running the jobs, and then rewards the caller by transferring back the spent gas plus some bonus as payment.

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

We observe from the previous code that, if the call to `_automationVault.exec()` fails, then the whole operation will fail. In this case, the caller will carry the gas costs of the failed transaction, without any payment.


Naturally, it is expected that callers will simulate the operation off-chain to avoid calling a job that they know will fail beforehand. However, this can be easily bypassed by a malicious job. The job could have different runtime behavior depending on whether it is called during the simulation or the effective on-chain transaction. This can be implemented either by contextual variables (such as timestamp, gas, block number, origin, etc) or by querying some storage variable (not necessarily in the job contract).

As opposed to other adapters such as the Keep3rRelay that have an underlying protocol that can eventually slash jobs (see https://docs.keep3r.network/core/jobs#disputing-and-slashing-a-job), the OpenRelay adapter doesn't provide such guarantees to the caller.

## Impact

Bots running with the OpenRelay can be griefed by making them spend gas in unsuccessful jobs without consequences for the job owner.

## Code Snippet

https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L12-L45

## Tool used

Manual Review

## Recommendation

The relay would require a redesign to consider the possibility of job failure without the keeper being responsible. In such cases, the keeper deserves compensation for the wasted effort, or the job owner should face consequences that discourage bad behavior.
