Virtual Felt Aphid

medium

# Loss to relayers as they would receive less payment for executing a job

## Summary

According to docs:

"*The OpenRelay is a fundamental yet potent relay system. It precisely calculates the gas used for executing your job within its smart contract, reimbursing the executor in ETH for the gas costs, plus an additional incentive.* 

*Current incentive: 120% of the gas costs, paid out in ETH instantaneously.*"

The `exec` function is designed to accept a relayed transaction with a transaction cost refund. At the beginning of the function, the `_initialGas` value stores the amount of gas that the relayer will approximately spend on the transaction initialization. Later inside the function, the total consumed gas is calculated as `_initialGas - gasleft()` and the appropriate refund is calculated including the bonus & is sent to the relayer.

User could provide calldata with zero padded bytes of arbitrary length. This would decrease the overall gas that is left thereby decreasing the refund that is to be paid to the relayer.

## Vulnerability Detail

Let’s say an attacker signs a transaction. Some of the relayers try to execute this transaction and send a transaction to the contract. Then, the attacker can frontruns the transaction, changing the transaction calldata by adding the zeroes bytes at the end.

So, the original transaction has such calldata:

```solidity
abi.encodeWithSignature("work()")
```
The modified (frontrun) transaction calldata:

```solidity
// Basically, just add zero bytes at the end
abi.encodeWithSignature("work()") || 0x00[]
```
This leads to an increased gas consumption which decreases the `_gasSpent` value.

```solidity
File: OpenRelay.sol
31:    uint256 _gasSpent = _initialGas - gasleft();
```
The `_payment`  value majorly depends on the `_gasSpent` (the rest being constants). Because the value of `_gasSpent` will be low in this case, it would lead to a decrease in the payment value that is to be sent to the relayer. 

```solidity
File: OpenRelay.sol
34:    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
```

The call would not revert inside AutomationVault contract because of the following check. It checks the initial bytes inside at the start of the jobData inside the `if` check but while calling, uses the entire data instead without conversion.

```solidity
File: AutomationVault.sol

413:      // Check that the selector is approved to be called
414:      if (!_approvedJobSelectors[msg.sender][_dataToExecute.job].contains(bytes4(_dataToExecute.jobData))) {
415:       revert AutomationVault_NotApprovedJobSelector();
416:      }
417:      (_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
```
## Impact
Loss to Relayers as they would receives less payment than intended. 
This may not benefit the attacker but to a relayer who executes the call, they would receive only a tiny portion of the refund than they had initially expected.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L31-L34

## Tool used
Manual Review

Link to a similar finding (not exactly the same impact because of diff code but for reference for the current submission) - [Link](https://code4rena.com/reports/2023-01-biconomy#h-02-theft-of-funds-under-relaying-the-transaction)

## Recommendation
It would be better to limit the length of the data to eliminate the possibility to put meaningless zeroes.