Jolly Sandstone Sealion

medium

# Gas griefing/theft is possible on unsafe external call

## Summary
This withdraw function  opens up a new attack-vector in the contract and it is gas griefing on the NATIVE_TOKEN transfer

## Vulnerability Detail
```bash
 (bool _success,) = _receiver.call{value: _amount}('');
```
Now (bool success, ) is actually the same as writing (bool success, bytes memory data) which basically means that even though the data is omitted it doesn’t mean that the contract does not handle it. Actually, the way it works is the bytes data that was returned from the receiver will be copied to memory. Memory allocation becomes very costly if the payload is big, so this means that if a receiver implements a fallback function that returns a huge payload, then the msg.sender of the transaction, in our case the relayer, will have to pay a huge amount of gas for copying this payload to memory.

## Impact
Malicious actor can launch a gas griefing attack on a relayer. Since griefing attacks have no economic incentive for the attacker and it also requires relayers it should be Medium severity.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L128

## Tool used

Manual Review

## Recommendation
Use a low-level assembly call since it does not automatically copy return data to memory

```solidity
bool success;
assembly {
    success := call(3000, receiver, amount, 0, 0, 0, 0)
}
```
