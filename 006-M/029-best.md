Bent Rosewood Swan

medium

# A vault can be exploited in multiple ways through the relays, especially if owner enables _ALL option

## Summary

Taking in consideration that according to the audit documentation all actions taken by a Relay besides the Gelato one are not trusted, we're going to explore a scenario in which a vault can be exploited by an Open Relay actor even if the jobs/function selectors are restricted and don't allow for direct funds theft.

This is especially true if the `_ALL` option has been allowed for the `_approvedCallers` for the Open Relay. 

## Vulnerability Detail

If the `_ALL` option is added to the `_approvedCallers`: 

```solidity

if (!_approvedCallers[msg.sender].contains(_relayCaller) && !_approvedCallers[msg.sender].contains(_ALL)) {
      revert AutomationVault_NotApprovedRelayCaller();
    }

```

It means that anyone can call the the `exec()` function on the Open Relay. Since the `msg.sender` is passed as an argument on the Open Relay to be the `_relayCaller`, if the `_ALL` options is enabled, it means that anyone will be able to perform actions on a certain vault through that relay. The way that the incentives are organized on the Open Relay, the actor calling the `exec()` function doesn't need any kind of an investment from their side (gas costs will be returned), i.e. they only get a reward for performing a certain action.

Even if the jobs / function selectors are heavily restricted not to allow any kind of funds transfer, they can still benefit from the reward, and since there's no restrictions to how many times a certain action can be performed, this can be abused.

For example, in the Open Relay the reward incentives are as follows:

```solidity

uint256 public constant GAS_BONUS = 53_000;
  /// @inheritdoc IOpenRelay
  uint256 public constant GAS_MULTIPLIER = 12_000;
  /// @inheritdoc IOpenRelay
  uint32 public constant BASE = 10_000;

```
```solidity

 // Execute the automation vault counting the gas spent
    uint256 _initialGas = gasleft();
    _automationVault.exec(msg.sender, _execData, new IAutomationVault.FeeData[](0));
    uint256 _gasSpent = _initialGas - gasleft();

    // Calculate the payment for the relayer
    uint256 _payment = (_gasSpent + GAS_BONUS) * block.basefee * GAS_MULTIPLIER / BASE;

_feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
    _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);

```

If the relay caller passed 120,000 gas, and the action took 80,000 to perform, it would mean that the `_gasSpent` would be 80,000.

If the block.basefee is 19 GWEI

According to the formula above, the actor would benefit:
(80_000 + 53_000) * 19000000000 * 12_000 / 10_000

3e15 or 0.003 ether
The transaction can be maliciously repeated to take a more significant amount of funds from the protocol. 

The Open Relay doesn't perform any checks as to who the caller is, and the same caller can also set an arbitrary address for the fee to which the funds would be directed.

This scenario doesn't presume that the approved caller would start to act maliciously, even though that is a plausible scenario as well.

Another exploitation plausibility involving the `_ALL` option could also work in terms of the Gelato Relay as well, even though that would not directly benefit the malicious actor. If the `_ALL` option is set for the Gelato Relay, anyone  would be able to call the `exec()` function an arbitrary amount of times. The addition to this is that the `exec()` function in the Gelato relay doesn't perform any checks whether execution data was provided. so even if jobs / selectors are restricted, a malicious user can pass an empty array as execution data and just forward fees from the protocol. 

This "attack" can be applicable to all vaults who have enabled `_ALL` as part of either the Open Relay and/or Gelato Relay.

## Impact
Repeated execution actions can cause financial damage to the protocol, especially if the `_ALL` option is enabled, allowing anyone to collect fees at no cost to them. 

## Code Snippet

https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/core/AutomationVault.sol#L399-L401

https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L34-L39

## Tool used

Manual Review

## Recommendation

Remove `_ALL` as an option for the trusted callers or at least only limit it to the Keep3r network as there are additional checks within the relays there. 
