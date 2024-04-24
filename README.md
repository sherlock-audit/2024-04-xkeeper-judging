# Issue M-1: L1 data fees are not reimbursed 

Source: https://github.com/sherlock-audit/2024-04-xkeeper-judging/issues/57 

## Found by 
IllIllI, Kirkeelee, Kose, LTDingZhen
## Summary

L1 data fees are not reimbursed, and they are often orders of magnitude more expensive than the L2 gas fees being reimbursed. Not reimbursing these fees will lead to jobs not being executed on L2s.


## Vulnerability Detail

While the contest README says that the protocol is interested in all EVM-compatible chains, its not clear to what extent they need compatibility. For instance, many chains have workarounds for the fact that they need to publish data from the L2 to the L1 via calldata, and therefore charge for those operations directly. Such caveats indicate that the compatibility is not 100% and therefore we can't really assume that any random chain will be supported. However, the README explicitly mentions Optimism as a target chain, so its idiosyncracies are in-scope.

The Gelato protocol [properly](https://docs.gelato.network/web3-services/relay/gelatos-fee-oracle#arguments-2) handles L1 data fees, but neither Keep3r nor OpenRelay do. It could be argued that for Keep3r, it's possible to modify a job's fee rate dynamically and therefore it's not that big a risk, but for OpenRelay, the reimbursement formula is hard-coded, which means there's no workaround.

Looking at a [transaction](https://optimistic.etherscan.io/tx/0x3c6aa4e1f5de25b14638147d6db893fdf8bff6a29dbe1fad86c93d6bfad2badd/advanced) from [this](https://xkeeper.network/optimism/vault/0xB1f5Ee0Ad3C469e9a88A77258dA5B56f9de2F219) vault, the transaction cost was 0.000305237594921218 ETH ($1.17) but only 0.00000008929694256 ETH (<$0.01) was reimbursed. The L1 data fee was 0.000304676890061656 Eth, whereas the gas fee was 0.000000111805735391.


## Impact

As is shown in [this](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/91#issuecomment-1704791659) analysis for another contest, the two fees do not have a fixed ratio, and the L1 data fee is frequently much larger than the L2 gas fee, for extended periods of time. This essentially means that the protocol is broken on L2s for any use case which requires timely executions. The contest README states that the sponsor is interested in issues where future integrations would be negatively impacted, and one such case would be where an exchange is trying to use xkeeper to handle customer operations, as is outlined at the end of the linked-to comment above. Operations will appear to hang for multiple hours at a time, causing loss of funds for customers trying to close their orders.


## Code Snippet

Only the L2 gas is reimbursed, not any of the L1 data fees:
```solidity
// File: solidity/contracts/relays/OpenRelay.sol : OpenRelay.exec()   #1

28        // Execute the automation vault counting the gas spent
29        uint256 _initialGas = gasleft();
30        _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
31        uint256 _gasSpent = _initialGas - gasleft();
32    
33        // Calculate the payment for the relayer
34        uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;
35    
36        // Send the payment to the relayer
37        IAutomationVault.FeeData[] memory _feeData = new IAutomationVault.FeeData[](1);
38        _feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
39:       _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);
```
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L28-L39


## Tool used

Manual Review


## Recommendation

Every L2 has its own formula for calculating the L1 data fee, so different versions of the code will have to be written for each L2. [This](https://docs.optimism.io/stack/transactions/fees#l1-data-fee) is the description for Optimism. Note that the Ecotone upgrade has occurred, so be sure to implement based on those or [these](https://specs.optimism.io/protocol/exec-engine.html#ecotone-l1-cost-fee-changes-eip-4844-da) instructions.




## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Kose** commented:
>  medium



# Issue M-2: Keepers can execute the `fallback` method even if it's not one of the approved selectors for that job. 

Source: https://github.com/sherlock-audit/2024-04-xkeeper-judging/issues/68 

## Found by 
Tricko
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



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**Kose** commented:
>  medium



**Hash01011122**

This issue is a borderline low/medium severity, as the occurrence involves selectors which typically are standard operations like `work()` or `exec()`. However, it may lean towards medium severity since it represents an edge case, due to its complexity.

