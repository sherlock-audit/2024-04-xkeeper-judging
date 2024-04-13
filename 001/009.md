Quick Sky Trout

high

# Malicious relay called could grief-call target contract and gain a gas reward

## Summary
Malicious relay executor could call relay's exec function before the automation vault admin removes him from active relay callers.

## Vulnerability Detail
The functionality the protocol offers is that a user can create an **automation vault** in which can be customized, like adding approved relays (calculate fees and call automation vault), callers (users with right to call relay) and  approved jobs (functions that a caller can execute). 

The admin of a vault could sends funds (native or erc-20) to the vault which are used to pay the caller. 

There are 3 supported relay providers the protocol supports - 2 of which are not trusted and also do not support time based or event based execution triggers - `Keep3rRelay` and `OpenRelay`.

The `OpenRelay` is created for custom bots (Ex: owned by the users of the protocol)  to call a job whenever needed and they get paid the gas used * 1,2 . The scenario is the same for the keep3r relay - the only difference being that the keeper relay is to execute the job once. 

Having all this in mind nothing stops a malicious `open` or `keep3r` relay executor to call the job until the funds in the vault are drained. Also the function called could have negative consequences on the target protocol. Even if the owner revokes the malicious caller from approved callers, he(the admin) could get front ran and the malicious caller could execute the target function at least once more, griefing and stealing fees.

Another issue here is that the owner can set a specific address as approved set in `Constants.sol`: 
`address constant _ALL = 0xFFfFfFffFFfffFFfFFfFFFFFffFFFffffFfFFFfF;`

This lets any user call automation vault through any approved relay and job selector, which leads to the same issue described above.
 ```solidity
function exec(address _relayCaller, ExecData[] calldata _execData, FeeData[] calldata _feeData) external {
     if (!_approvedCallers[msg.sender].contains(_relayCaller) && !_approvedCallers[msg.sender].contains(_ALL)) {
      revert AutomationVault_NotApprovedRelayCaller();
    }
...
}
 ```
## Impact
At worst case the funds from the automation vault get drained and the job being called creates havoc for the target protocol

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L399

## Tool used
Manual Review

## Recommendation
Add a flag that if set to true pauses the `AutomationVault::exec` function after a call so the owner can revoke permissions of the caller without being frontran.  Or add a mechanism to limit the number, frequency and or payment of requests for the relays that could host malicious callers.