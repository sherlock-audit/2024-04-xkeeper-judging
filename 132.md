Dry Mint Koala

medium

# Keep3r Relay Implementations are Not Compatible with Keep3r in Optimism and Executions Will Always Revert

## Summary
`Keep3rRelay` and `Keep3rBondedRelay` uses deprecated function for sidechains and `exec` calls will always revert.
## Vulnerability Detail
Keep3r takes different arguments for function `worked()` in sidechains in order to estimate gas usage and rewards for keepers properly:
```solidity
  /// @dev Sidechain implementation deprecates worked(address) as it should come with a usdPerGasUnit parameter
  function worked(address) external pure override {
    revert Deprecated();
  }

  /// @notice Implemented by jobs to show that a keeper performed work
  /// @dev Uses a USD per gas unit payment mechanism
  /// @param _keeper Address of the keeper that performed the work
  /// @param _usdPerGasUnit Units of USD (in wei) per gas unit that should be rewarded to the keeper
  function worked(address _keeper, uint256 _usdPerGasUnit) external override {
```
The snippet above taken from [Keep3rSidechain.sol](https://optimistic.etherscan.deth.net/address/0x745a50320B6eB8FF281f1664Fc6713991661B129#code) that is live in optimism currently. We can also see that this contract is exact contract that will be interacted as it is the address of `KEEPER_V2` in deployed Keep3r Relays by xKeeper in Optimism. Deployed addresses can checked from [here](https://docs.xkeeper.network/content/intro/index.html)
But relay contracts implemented by xKeeper uses the `worked()` function that is deprecated:
```solidity
    // Inject the final call which will issue the payment to the keeper
    _execDataKeep3r[_execDataLength + 1] = IAutomationVault.ExecData({
      job: address(KEEP3R_V2),
      jobData: abi.encodeWithSelector(IKeep3rV2.worked.selector, msg.sender)
    });
```
Hence in chains other than mainnet, Keep3r calls will always revert.
## Impact
Current Keep3r Relay contracts are not compatible with Keep3r in Optimism (Keeper is only deployed to Mainnet and Optimism currently). Although vault creation will succeed, `exec()` called by keepers will always revert.
## Code Snippet
[Keep3rRelay.sol](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/Keep3rRelay.sol/#L52-L55)
[Keep3rBondedRelay.sol](https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/Keep3rBondedRelay.sol/#L76-L80)
## Tool used

Manual Review

## Recommendation
Either implement a compatible version for Keep3rSideChain, or don't use Keep3r in sidechains.