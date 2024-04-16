Colossal Lead Woodpecker

medium

# Function `values` imported from `EnumerableSet` will cause out of gas issues

## Summary

If there are too many approved callers in the mapping `_approvedCallers`, within the function `modifyRelayCallers` the call to the function `values` will cause a revert due to it being out of gas.

## Vulnerability Detail

The `AutomationVault` contract imports OpenZeppelin's [EnumerableSet](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol), which adds additional types and functions to help support the management of sets within the contract. One of the added functions is [values](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L285-L303) which takes a set as an argument and returns the stored addresses in an array. As stated in the NatSpec of the function: `This operation will copy the entire storage to memory, which can be quite expensive.`. If the approved users are so much so that the call reverts due to it being out of gas, modifyRelayCallers's functionality is completely broken due to it reverting at the beginning of each call. (which can be seen in the PoC provided below). If the approved users are >5000, the gas cost to call `values` would be ~10 million, which won't be enough to revert the function, but the revert will happen by the time the for loop, which removes callers, ends.

## Impact

Medium - As the owner of the vault wouldn't be able to remove/add approved callers to the relay, which is one of the core functionalities of the contract.

## Code Snippet

### modifyRelayCallers

```solidity
  function modifyRelayCallers(address _relay, address[] memory _callers) public onlyOwner {
    if (_relay == address(0)) revert AutomationVault_RelayZero();
    // Create the counter variable
    uint256 _i;

    // Get the approved callers
    address[] memory _oldApprovedCallers = _approvedCallers[_relay].values();

    // Get the length of the approved callers array
    uint256 _callersLength = _oldApprovedCallers.length;

    // Remove the callers
    for (_i; _i < _callersLength;) {
      _approvedCallers[_relay].remove(_oldApprovedCallers[_i]);

      unchecked {
        ++_i;
      }
    }

    // Get the length of the callers array
    _callersLength = _callers.length;

    // Set the callers for the relay
    for (_i = 0; _i < _callersLength;) {
      if (_approvedCallers[_relay].add(_callers[_i])) {
        emit ApproveRelayCaller(_relay, _callers[_i]);
      }

      unchecked {
        ++_i;
      }
    }
  }
```

### Values

```solidity
    function values(AddressSet storage set) internal view returns (address[] memory) {
        bytes32[] memory store = _values(set._inner);
        address[] memory result;

        /// @solidity memory-safe-assembly
        assembly {
            result := store
        }

        return result;
    }
```

## Proof of Concept

Add the following code snippet in: `test/unit/AutomationVault.t.sol` in the contract `UnitAutomationVaultModifyRelayCallers`:

<details>

```solidity
  mapping(address _relay => EnumerableSet.AddressSet _callers) internal _approvedCallersTest;
  address[] public _callers;
    
    // from openzeppelin-contracts/contract/utils/Strings.sol 
    function toString(uint256 value) internal pure returns (string memory) {
    if (value == 0) {
      return '0';
    }

    uint256 temp = value;
    uint256 digits;

    while (temp != 0) {
      digits++;
      temp /= 10;
    }

    bytes memory buffer = new bytes(digits);

    while (value != 0) {
      digits--;
      buffer[digits] = bytes1(uint8(48 + (value % 10)));
      value /= 10;
    }

    return string(buffer);
  }
  
   // from openzeppelin-contracts/contract/utils/Strings.sol 
  function convertToString(uint256 x) public pure returns (string memory) {
    string memory str = toString(x);
    return str;
  }
  
  function setUp() public virtual override {
    vm.pauseGasMetering();
    _hardCodedRelay = makeAddr('_relay');
    uint256 numOfCallers = 5000;

    for (uint256 j; j < numOfCallers; ++j) {
      vm.pauseGasMetering();
      string memory user = convertToString(j);
      _callers.push(makeAddr(user));
    }

    _assumeRelayData(_hardCodedRelay, _callers, makeAddr('job'), new bytes4[](1));
    automationVault = new AutomationVaultForTest(owner, _NATIVE_TOKEN);

    automationVault.addRelayForTest(_hardCodedRelay, new address[](0), address(0), new bytes4[](0));

    for (uint256 _i; _i < _callers.length; ++_i) {
      _approvedCallersTest[_hardCodedRelay].add(_callers[_i]);
    }
  }
  
  function testTooManyApprovedCallers() public {
    vm.resumeGasMetering();
    console.log('Gas before values call:', gasleft());
    address[] memory newApprovedCallers = _approvedCallersTest[_hardCodedRelay].values();
    console.log('newApprovedCallers length:', newApprovedCallers.length);
    console.log('Gas after values call:', gasleft());
  }

```
</details>

## Tool used

Manual Review

## Recommendation

One suggestion that I have is to loop through parts of the set and add the number of `approvedCallers` to a variable within the function, that way, it won't be done in one big call to `values` but in a few smaller ones.