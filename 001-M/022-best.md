Sleepy Quartz Marmot

medium

# `OpenRelay` will be unoperatable if `NATIVE_TOKEN` in `AutomationVault` is set to other values.

## Summary
When an user deploys the vault with some other values other than `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, all [`exec`](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol#L21) to [`OpenRelay`](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/OpenRelay.sol) will fail.

## Vulnerability Detail
In `OpenRelay` contract, which is open to all bots, fees are distributed in native tokens. This value is imported from `Constants.sol` and states the `_NATIVE_TOKEN` address to be `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`.

```solidity

    // Send the payment to the relayer
    IAutomationVault.FeeData[] memory _feeData = new IAutomationVault.FeeData[](1);
    _feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
    // @audit re-entrancy here
    _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);
```

Then, these fee data will be included and altogether send to `AutomationVault` for further execution. Similarily, in `AutomationVault`, fees are sent after all executions are successfully done:

```solidity
    for (_i; _i < _dataLength;) {
      _feeInfo = _feeData[_i];

      // If the token is the native token, transfer the funds to the receiver, otherwise transfer the tokens
      if (_feeInfo.feeToken == NATIVE_TOKEN) {
        // @note re-entrancy?
        (_success,) = _feeInfo.feeRecipient.call{value: _feeInfo.fee}('');
        if (!_success) revert AutomationVault_NativeTokenTransferFailed();
      } else {
        IERC20(_feeInfo.feeToken).safeTransfer(_feeInfo.feeRecipient, _feeInfo.fee);
      }

      // Emit the event
      emit IssuePayment(msg.sender, _relayCaller, _feeInfo.feeRecipient, _feeInfo.feeToken, _feeInfo.fee);

      unchecked {
        ++_i;
      }
```

The logic is quite simple, it checks if the incoming token is `NATIVE_TOKEN` or not, if it's indeed native token, transfers the correspond Ethers to recipient, otherwise, treat the token as ERC20, and then transfer.

The problem is, this `NATIVE_TOKEN` variable is set in constructor, and not a constant imported from other contracts:

```solidity
  constructor(address _owner, address _nativeToken) {
    owner = _owner;
    NATIVE_TOKEN = _nativeToken;
  }
```

We can also see this is indeed a parameter in the factory contract:

```solidity
  function deployAutomationVault(
    address _owner,
    address _nativeToken,
    uint256 _salt
  ) external returns (IAutomationVault _automationVault) {
    // Create the new automation vault with the owner
    _automationVault = new AutomationVault{salt: keccak256(abi.encodePacked(msg.sender, _salt))}(_owner, _nativeToken);

    if (address(_automationVault) == address(0)) revert AutomationVaultFactory_Create2Failed();

    // Add the automation vault to the list of automation vaults
    _automationVaults.add(address(_automationVault));

    // Emit the event
    emit DeployAutomationVault(_owner, address(_automationVault));
  }
```

Whereas, this `_NATIVE_TOKEN` is imported from a contract, and hence a fixed value. When there is a discrepancy between `NATIVE_TOKEN` and `_NATIVE_TOKEN`, the vault's logic will treat it as:

1. The token address is not `NATIVE_TOKEN` set in the constructor, so this must be an ERC20 token.
2. Proceeds to treat this address as ERC20 token, and tries call the `transfer` function
3. This will fail because `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` is likely not a contract, or a valid ERC20 token


## Impact
Suppose Bob deploys such vault with native token being the zero address. As a result of this issue, all interactions in `OpenRelay` will not work because:

1. `OpenRelay` sets the token address to be `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`
2. The vault interprets the address and finds out it's not native token because it's not `address(0)`
3. The vault tries to call `IERC20(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE).safeTransfer()`, and fails
4. And all other interactions will also fail, because `NATIVE_TOKEN` is immutable, Bob may have to deploy another vault and make sure the native token address is set to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`.

A coded PoC:

```solidity
    function test_NativeTokenRevert() public {
        vm.stopPrank();

        address bob = address(0x1337);
        IAutomationVault vault = automationVaultFactory.deployAutomationVault(bob, address(0), 0);
        assertEq(vault.NATIVE_TOKEN(), address(0));
        assertEq(vault.owner(), bob);
        address[] memory _bots = new address[](1);
        _bots[0] = bot;

        // Job selectors array
        bytes4[] memory _jobSelectors = new bytes4[](2);
        _jobSelectors[0] = basicJob.work.selector;
        _jobSelectors[1] = basicJob.workHard.selector;

        // Job data array
        IAutomationVault.JobData[] memory _jobsData = new IAutomationVault.JobData[](1);
        _jobsData[0] = IAutomationVault.JobData(address(basicJob), _jobSelectors);

        vm.startPrank(bob);

        // AutomationVault approve relay data
        vault.addRelay(address(openRelay), _bots, _jobsData);
        address(vault).call{value: 100 ether}('');

        vm.stopPrank();
        vm.startPrank(bot);
        IAutomationVault.ExecData[] memory _execData = new IAutomationVault.ExecData[](1);
        _execData[0] =
        IAutomationVault.ExecData(address(basicJob), abi.encodeWithSelector(basicJob.workHard.selector, 1));
        openRelay.exec(vault, _execData, bot);

    }
```
This will revert because it tries to call a non-contract.

Since the vault has provided a way for owners to withdraw tokens out, so a fail deployment can always be resolved quite easily, but before the user notices the issue, all the interactions set to bots will fail. Consider it can make one relay not functional but with a rather easy way to resolve, I believe this should be medium severity.

## Code Snippet
```solidity
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

    // Send the payment to the relayer
    IAutomationVault.FeeData[] memory _feeData = new IAutomationVault.FeeData[](1);
    _feeData[0] = IAutomationVault.FeeData(_feeRecipient, _NATIVE_TOKEN, _payment);
    _automationVault.exec(msg.sender, new IAutomationVault.ExecData[](0), _feeData);

    // Emit the event
    emit AutomationVaultExecuted(_automationVault, msg.sender, _execData, _feeData);
  }
```


## Tool used

Manual Review, foundry

## Recommendation
Either set native token in the vault to a fixed value like `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, or modify the code in `OpenRelay.exec` as:
```solidity
_feeData[0] = IAutomationVault.FeeData(_feeRecipient, _automationVault.NATIVE_TOKEN(), _payment);
```
