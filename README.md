# Project: Create a Token and Balance Conversion

In this Solidity and React project, the task involves creating a smart contract using the Remix environment and a frontend application using Next.js. The goal is to establish a basic token that can facilitate minting, burning, and balance doubling operations. Additionally, the frontend application will display the balance in both ETH and USD.

## Description

This program demonstrates how to create a token using Solidity and a frontend application to interact with the smart contract. The smart contract includes minting, burning, and doubling balance functionalities. The frontend application displays the balance in both ETH and USD.

## Getting Started

### Prerequisites

- **Remix**: Use the Remix website at [Remix Ethereum](https://remix.ethereum.org/).
- **Node.js and npm**: Ensure you have Node.js and npm installed for the frontend application.

### Installing

1. **Frontend Application Setup**

   ```bash
   npx create-next-app my-token-app
   cd my-token-app
   npm install ethers axios
   
2. **Solidity Contract**

  Open Remix and create a new file named Assessment.sol.
  Copy and paste the Solidity code provided below.
  
## Contract: Assessment.sol
```// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

contract Assessment {
    address payable public owner;
    uint256 public balance;

    event Deposit(uint256 amount);
    event Withdraw(uint256 amount);
    event BalanceDoubled(uint256 newBalance);

    constructor(uint initBalance) payable {
        owner = payable(msg.sender);
        balance = initBalance;
    }

    function getBalance() public view returns(uint256) {
        return balance;
    }

    function deposit(uint256 _amount) public payable {
        uint _previousBalance = balance;

        // make sure this is the owner
        require(msg.sender == owner, "You are not the owner of this account");

        // perform transaction
        balance += _amount;

        // assert transaction completed successfully
        assert(balance == _previousBalance + _amount);

        // emit the event
        emit Deposit(_amount);
    }

    // custom error
    error InsufficientBalance(uint256 balance, uint256 withdrawAmount);

    function withdraw(uint256 _withdrawAmount) public {
        require(msg.sender == owner, "You are not the owner of this account");
        uint _previousBalance = balance;
        if (balance < _withdrawAmount) {
            revert InsufficientBalance({
                balance: balance,
                withdrawAmount: _withdrawAmount
            });
        }

        // withdraw the given amount
        balance -= _withdrawAmount;

        // assert the balance is correct
        assert(balance == (_previousBalance - _withdrawAmount));

        // emit the event
        emit Withdraw(_withdrawAmount);
    }

    function doubleBalance() public {
        require(msg.sender == owner, "You are not the owner of this account");
        balance *= 2;

        // emit the event
        emit BalanceDoubled(balance);
    }
}
```
# Frontend Application: pages/index.js

```
import { useState, useEffect } from "react";
import { ethers } from "ethers";
import axios from "axios"; // Import Axios
import atm_abi from "../artifacts/contracts/Assessment.sol/Assessment.json";

export default function HomePage() {
  const [ethWallet, setEthWallet] = useState(undefined);
  const [account, setAccount] = useState(undefined);
  const [atm, setATM] = useState(undefined);
  const [balance, setBalance] = useState(undefined);
  const [usdBalance, setUsdBalance] = useState(undefined); // State for USD balance

  const contractAddress = "0x5FbDB2315678afecb367f032d93F642f64180aa3";
  const atmABI = atm_abi.abi;

  // Function to fetch ETH to USD conversion rate
  const fetchEthToUsdRate = async () => {
    try {
      const response = await axios.get(
        "https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd"
      );
      const ethUsdRate = response.data.ethereum.usd;
      return ethUsdRate;
    } catch (error) {
      console.error("Error fetching ETH to USD rate:", error);
      return null;
    }
  };

  const getWallet = async () => {
    if (window.ethereum) {
      setEthWallet(window.ethereum);
    }

    if (ethWallet) {
      const accounts = await ethWallet.request({ method: "eth_accounts" });
      handleAccount(accounts);
    }
  };

  const handleAccount = (accounts) => {
    if (accounts.length > 0) {
      console.log("Account connected: ", accounts[0]);
      setAccount(accounts[0]);
    } else {
      console.log("No account found");
    }
  };

  const connectAccount = async () => {
    if (!ethWallet) {
      alert("MetaMask wallet is required to connect");
      return;
    }

    const accounts = await ethWallet.request({ method: "eth_requestAccounts" });
    handleAccount(accounts);

    // once wallet is set we can get a reference to our deployed contract
    getATMContract();
  };

  const getATMContract = () => {
    const provider = new ethers.providers.Web3Provider(ethWallet);
    const signer = provider.getSigner();
    const atmContract = new ethers.Contract(contractAddress, atmABI, signer);

    setATM(atmContract);
  };

  const getBalance = async () => {
    if (atm) {
      const ethBalance = await atm.getBalance();
      const ethBalanceNumber = ethers.utils.formatEther(ethBalance);
      setBalance(Number(ethBalanceNumber)); // Convert to Number to remove decimals

      // Fetch ETH to USD rate
      const ethUsdRate = await fetchEthToUsdRate();
      if (ethUsdRate) {
        const usdValue = ethBalanceNumber * ethUsdRate;
        setUsdBalance(usdValue.toFixed(2)); // Convert to USD and round to 2 decimal places
      }
    }
  };

  const deposit = async () => {
    if (atm) {
      let tx = await atm.deposit(ethers.utils.parseEther("1"));
      await tx.wait();
      getBalance();
    }
  };

  const withdraw = async () => {
    if (atm) {
      let tx = await atm.withdraw(ethers.utils.parseEther("1"));
      await tx.wait();
      getBalance();
    }
  };

  const doubleBalance = async () => {
    if (atm) {
      let tx = await atm.doubleBalance();
      await tx.wait();
      getBalance();
    }
  };

  const initUser = () => {
    // Check to see if user has Metamask
    if (!ethWallet) {
      return <p>Please install Metamask in order to use this ATM.</p>;
    }

    // Check to see if user is connected. If not, connect to their account
    if (!account) {
      return <button onClick={connectAccount}>Please connect your Metamask wallet</button>;
    }

    if (balance === undefined) {
      getBalance();
    }

    return (
      <div>
        <p>Your Account: {account}</p>
        <p>Your Balance: {balance} ETH ({usdBalance ? `$${usdBalance}` : "Loading..."})</p>
        <button onClick={deposit}>Deposit 1 ETH</button>
        <button onClick={withdraw}>Withdraw 1 ETH</button>
        <button onClick={doubleBalance}>Double Balance</button>
      </div>
    );
  };

  useEffect(() => {
    getWallet();
  }, []);

  return (
    <main className="container">
      <header>
        <h1>Welcome to the Metacrafters ATM!</h1>
      </header>
      {initUser()}
      <style jsx>{`
        .container {
          text-align: center;
        }
      `}</style>
    </main>
  );
}
```
# Explanation of Added Features
1. ## ETH to USD Conversion

  Function: fetchEthToUsdRate
    This function fetches the current conversion rate of ETH to USD from the CoinGecko API.
    It uses Axios to make an HTTP GET request to the API endpoint.
    The conversion rate is extracted from the response data and returned.
    
  Usage:
    The getBalance function calls fetchEthToUsdRate to obtain the conversion rate.
    It then calculates the USD value of the ETH balance and updates the usdBalance state.
    
2. ## Doubling Balance

  Function: doubleBalance in Solidity
    This function doubles the current balance of the contract.
    It ensures that only the owner of the contract can call this function.
    The function emits a BalanceDoubled event after successfully doubling the balance.

  ```
    function doubleBalance() public {
  require(msg.sender == owner, "You are not the owner of this account");
  balance *= 2;

  // emit the event
  emit BalanceDoubled(balance);
}
```

Usage:
The doubleBalance function in the frontend calls the doubleBalance function of the smart contract.
It waits for the transaction to be confirmed and then updates the balance
```
const doubleBalance = async () => {
  if (atm) {
    let tx = await atm.doubleBalance();
    await tx.wait();
    getBalance();
  }
};
```
## Authors
Vashti Leonie D. Bauson

## License
This project is licensed under the MIT License - see the LICENSE.md file for details
