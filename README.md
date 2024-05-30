# Smart-Contract-Management: Function Frontend
This Solidity smart contract create a simple contract with 2-3 functions. Then show the values of those functions in
frontend of the application.

## Description
Good day! My name is MARC and We are here again for another poroject assessment in Metacrafters. This assessment I will discuss
how to create a simple contract and show the values of those functions in frontend of the application.
## Getting Started
          //Assesment.sol (code)
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

contract Assessment {
    address payable public owner;
    uint256 public balance;

    event Deposit(uint256 amount, address from);
    event Withdraw(uint256 amount, address to);
    event Transfer(address to, uint256 amount);

    constructor(uint initBalance) payable {
        owner = payable(msg.sender);
        balance = initBalance;
    }

    function getBalance() public view returns (uint256) {
        return balance;
    }

    function deposit(uint256 _amount) public payable {
        uint _previousBalance = balance;

        require(msg.sender == owner, "You are not the owner of this account");
        require(_amount > 0 && _amount <= 10, "Deposit amount must be between 1 and 10 tokens");

        balance += _amount;

        assert(balance == _previousBalance + _amount);

        emit Deposit(_amount, msg.sender);
    }

    function withdraw(uint256 _amount) public {
        require(msg.sender == owner, "You are not the owner of this account");
        require(_amount > 0 && _amount <= balance, "Invalid withdrawal amount");

        balance -= _amount;
        owner.transfer(_amount);

        emit Withdraw(_amount, msg.sender);
    }

function transfer(address payable _to, uint256 _amount) public {
    require(msg.sender == owner, "You are not the owner of this account");
    require(_amount > 0 && _amount <= 10, "Transfer amount must be between 1 and 10 tokens");
    require(balance >= _amount, "Insufficient balance");

    // Transfer tokens
    balance -= _amount;
    _to.transfer(_amount);

    // Calculate and transfer equivalent Ethereum amount, limited to 10 times the token amount
    uint256 ethAmount = _amount * 1 ether;
    if (ethAmount > 10 ether) {
        ethAmount = 10 ether;
    }
    _to.transfer(ethAmount);

    emit Transfer(_to, _amount);
    }
}
        //Index.js (code)
        import { useState, useEffect } from "react";
import { ethers } from "ethers";
import atm_abi from "../artifacts/contracts/Assessment.sol/Assessment.json";

export default function HomePage() {
  const [ethWallet, setEthWallet] = useState(undefined);
  const [account, setAccount] = useState(undefined);
  const [atm, setATM] = useState(undefined);
  const [balance, setBalance] = useState(undefined);
  const [showAddress, setShowAddress] = useState(true);
  const [amount, setAmount] = useState("");
  const [transferAmount, setTransferAmount] = useState("");
  const [recipient, setRecipient] = useState("");
  const [errorMessage, setErrorMessage] = useState("");
  const [showError, setShowError] = useState(false);

  const contractAddress = "0x5FbDB2315678afecb367f032d93F642f64180aa3";
  const atmABI = atm_abi.abi;

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
    if (accounts && accounts.length > 0) {
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
      setBalance((await atm.getBalance()).toNumber());
    }
  };

  const deposit = async () => {
    if (atm && amount) {
      const depositAmount = parseInt(amount);
      if (depositAmount > 0 && depositAmount <= 10) {
        let tx = await atm.deposit(depositAmount, { value: ethers.utils.parseEther(amount) });
        await tx.wait();
        setBalance(balance + depositAmount);
        setShowError(false);
      } else {
        setErrorMessage("You can only deposit up to 10 Tokens only");
        setShowError(true);
      }
    }
  };

  const withdraw = async () => {
    if (atm && amount) {
      const withdrawAmount = parseInt(amount);
      if (withdrawAmount > 0 && withdrawAmount <= balance) {
        let tx = await atm.withdraw(withdrawAmount);
        await tx.wait();
        setBalance(balance - withdrawAmount);
        setShowError(false);
      } else {
        setErrorMessage("You can only withdraw up to 10 Tokens only");
        setShowError(true);
      }
    }
  };

  const transfer = async () => {
    if (atm) {
      if (!transferAmount && !recipient) {
        setErrorMessage("Please enter a value first");
        setShowError(true);
        return;
      }
  
      if (!transferAmount) {
        setErrorMessage("Please enter an amount of up to 10 tokens");
        setShowError(true);
        return;
      }
  
      if (!recipient) {
        setErrorMessage("Please enter a valid address (e.g., 0x5FbDB2315678afecb367f032d93F642f64180aa3)");
        setShowError(true);
        return;
      }
  
      const amountToTransfer = parseInt(transferAmount);
      if (amountToTransfer > 0 && amountToTransfer <= 10 && ethers.utils.isAddress(recipient)) {
        try {
          let tx = await atm.transfer(recipient, amountToTransfer);
          await tx.wait();
          setBalance(balance - amountToTransfer);
          setShowError(false);
        } catch (error) {
          setErrorMessage("Transfer failed");
          setShowError(true);
        }
      } else {
        setErrorMessage("Please enter a valid recipient address and amount");
        setShowError(true);
      }
    }
  };  

  const toggleAddressVisibility = () => {
    setShowAddress(!showAddress);
  };

  const handleAmountChange = (e) => {
    setAmount(e.target.value);
  };

  const handleTransferAmountChange = (e) => {
    setTransferAmount(e.target.value);
  };

  const handleRecipientChange = (e) => {
    setRecipient(e.target.value);
  };

  const initUser = () => {
    if (!ethWallet) {
      return <p>Please install MetaMask in order to use this wallet.</p>;
    }

    if (!account) {
      return <button onClick={connectAccount}>Open Wallet</button>;
    }

    if (balance === undefined) {
      getBalance();
    }

    return (
      <div className="wallet-container">
        <button onClick={toggleAddressVisibility}>Show or Hide my Address</button>
        {showAddress && <p>My Current Address: {account}</p>}
        <p>Current Balance: {balance}</p>
        <div className="buttons-container">
          <input type="number" value={amount} onChange={handleAmountChange} placeholder="Enter amount" min="1" max="10" />
          <button onClick={deposit}>Deposit</button>
          <button onClick={withdraw}>Withdraw</button>
          <input type="number" value={transferAmount} onChange={handleTransferAmountChange} placeholder="Enter transfer amount" min="1" max="10" />
          <input type="text" value={recipient} onChange={handleRecipientChange} placeholder="Enter recipient address" />
          <button onClick={transfer}>Transfer</button>
        </div>
        {showError && <div className="error-message">{errorMessage}</div>}
      </div>
    );
  };

  useEffect(() => {
    getWallet();
  }, []);

  return (
    <main className="container">
  <header>
    <h1>MY META WALLET</h1>
  </header>
  {initUser()}
  <div className="wallet-container">
    {/* Your wallet content goes here */}
    <button className="author-button">By MAMARC O2</button>
  </div>
  <style jsx>{`
    .container {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      padding: 20px;
      background: linear-gradient(45deg, #2e2e2e, #383838);
      color: #fff;
      font-family: 'Arial, sans-serif';
    }

    header {
      text-align: center;
      margin-bottom: 40px;
    }

    header h1 {
      font-size: 3em;
      margin-bottom: 10px;
      text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
    }

    .wallet-container {
      border: 2px solid #4caf50;
      padding: 20px;
      border-radius: 25px;
      background-color: #212121;
      text-align: center;
      margin: 20px auto;
      width: 320px;
      position: relative;
      box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3);
      transition: transform 0.3s, box-shadow 0.3s;
    }

    .wallet-container:hover {
      transform: translateY(-10px);
      box-shadow: 0 12px 24px rgba(0, 0, 0, 0.5);
    }

    .author-button {
      margin-top: 20px;
      padding: 10px 20px;
      border: none;
      border-radius: 20px;
      background-color: #4caf50;
      color: #fff;
      font-size: 16px;
      cursor: pointer;
      transition: background-color 0.3s, transform 0.3s;
    }

    .author-button:hover {
      background-color: #45a049;
      transform: scale(1.05);
    }

    .error-message {
      color: red;
      position: absolute;
      top: -20px;
      left: 50%;
      transform: translateX(-50%);
      background-color: #fff;
      padding: 5px 15px;
      border: 1px solid red;
      border-radius: 5px;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }
  `}</style>
</main>
 );
}
  
        
## Executing Program
To run this program, you can use Remix, an online Solidity IDE. To get started, go to the Remix website at https://remix.ethereum.org/. 
Once you are on the Remix website, create a new file by clicking on the "+" icon in the left-hand sidebar. Save the file with a .sol extension 
(e.g., HelloWorld.sol). Then copy the code given in the assessment.

## Author
Marc Danniel Mariazeta 61902476@ntc.edu.ph
