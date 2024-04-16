Docile Strawberry Tarantula

high

# Payable Jobs that require sending NATIVE tokens can not be executed

## Summary

## Vulnerability Detail
Relays call [AutomationVault::exec](https://github.com/sherlock-audit/2024-04-xkeeper/blob/b82de26633c1904ccfb3c121497845128f1c5c5f/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L397-L452) to execute jobs allowed to them. A low-level call is made to interact with the target contract and execute the job
```js
(_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
if (!_success) revert AutomationVault_ExecFailed();
```
We can notice here that for payable jobs functions, their execution **will ALWAYS revert**. Notice that even if the contract holds enough native token balance, the transaction will still revert as the value needed by the payable job function is not passed.
## Impact
- Payable functions can not be executed.
## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/b82de26633c1904ccfb3c121497845128f1c5c5f/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L417
## Tool used

Manual Review

## Recommendation
Consider adding a new field in `ExecData` to register the native token amount needed by the job, and pass it when making the call. Below is a suggestion for an updated code.

`IAutomationVault`
```diff
/**
   * @notice The data to execute a job
   * @param job The address of the job
   * @param jobData The data to execute the job
+  * @param nativeTokenAmount the native token amount needed to execute the job
   */
  struct ExecData {
    address job;
    bytes jobData;
+   uint256 nativeTokenAmount 
  }
```
`AutomationVault`
```diff
/// @inheritdoc IAutomationVault
function exec(address _relayCaller, ExecData[] calldata _execData, FeeData[] calldata _feeData) external {
    // rest of the code
-    (_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
+    if (dataToExecute.nativeTokenAmount > 0) {
+    (_success,) = _dataToExecute.job.call{value: _dataToExecute.nativeTokenAmount}(_dataToExecute.jobData);
+    } else {
+     (_success,) = _dataToExecute.job.call(_dataToExecute.jobData);
+    }
    if (!_success) revert AutomationVault_ExecFailed();

    // rest of the code
}
```