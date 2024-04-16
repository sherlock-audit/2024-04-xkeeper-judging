Acidic Azure Deer

medium

# OOG scenario can occur in multiple AutomationVault functions

## Summary

Most functions in the AutomationVault.sol contract use multiple for loops with nesting to perform add() and remove() operations on callers, jobs and selectors data of a relay.

## Vulnerability Detail

The functions that would be mainly affected are:
1. [deleteRelay()](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L207)
2. [modifyRelayJobs()](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L319)

The second function modifyRelayJobs() mentioned above is more gas intensive since it includes the use of 4 for loops i.e. 2 nested loops. The function modifies the data associated with a relay by first deleting the existing data (old jobs and selectors) and then adding the new data (new jobs and selectors). Both processes would use up more gas as the owner increases size of jobs and selectors causing higher number of iterations to occur.

## Impact

The issue is that if the number of callers, jobs or selectors for a particular relay increase in size, there is a high chance that deletions and modifications would revert, thus making the owner devoid of changing any data. 

## Code Snippet

How to use this POC:
 - Add the POC to AutomationVault.t.sol in the `test/unit` folder within the `UnitAutomationVaultModifyRelayJobs` contract located in the file.
 - Run the POC using `forge test --match-test testRelayJobsAndSelectorsAreModifiedIssue -vvvvv`
 - The gas used by the modifyRelayJobs() function call would be around 15 million (estimations vary quite a bit in foundry). If we bump up the numbers by a bit in the for loop of the POC, we'll encounter an OOG exception causing a revert. In general, the current gas used up is extremely high as well since it's way higher than the average gas limit.
```solidity
  function testRelayJobsAndSelectorsAreModifiedIssue(
    address _relay,
    address _job,
    bytes4[] memory _selectors
  ) public happyPath(_relay, _job, _selectors) {
    // Get the list of jobs data for the relay
    (, IAutomationVault.JobData[] memory _jobsData) = automationVault.getRelayDataForTest(_relay);

    // Length should be zero
    assertEq(_jobsData.length, 0);

    //_jobsData = _createJobsData(_job, _selectors);

    _jobsData = new IAutomationVault.JobData[](100);
    for (uint256 i; i < 100 ; i++) {
      _jobsData[i].job = _job;
      _jobsData[i].functionSelectors = _selectors;
    }

    // Modify the relay jobs
    automationVault.modifyRelayJobs(_relay, _jobsData);

    // Get the list of jobs data for the relay
    (, _jobsData) = automationVault.getRelayDataForTest(_relay);

    for (uint256 _i; _i < _jobsData.length; ++_i) {
      assertEq(_jobsData[_i].job, _job);
      assertEq(_jobsData[_i].functionSelectors.length, _cleanSelectors.length());

      for (uint256 _j; _j < _jobsData[_i].functionSelectors.length; ++_j) {
        assertEq(_jobsData[_i].functionSelectors[_j], _cleanSelectors.at(_j));
      }
    }
  }
```

## Tool used

Manual Review

## Recommendation

Allow the owner to delete old jobs and selectors in a separate function and introduce a new function to allow owner to update new jobs and selectors for the relay. This is the most simplistic solution I could think of though there are many ways to go about this e.g. introducing start and end indexes to allow owner to perform operations in batches.