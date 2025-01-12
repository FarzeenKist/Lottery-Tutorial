# Introduction
- Randomness is a core functionality for many applications
- Games, trait-based NFTs, and luck-based financial applications rely heavily on randomness
- Welcome! In this tutorial, we are going to learn about random number generation and Working with it
---
# Pre requisites
- To understand and utilize this tutorial you need to have the understanding of:
    - Basic knowledge of developing smart contracts in Solidity
    - Basic experience with Remix IDE 
    - Working with Celo Alfajores testnet
---
# Requirements
- Metamask
- Celo Alfajores testnet
---
# Let's Get started!
## What we are going to build
- A lottery contract where users can join the lottery by paying an entry amount (just like buying a lottery in real life)
- A random winner will be chosen from the players and will be rewarded the entire lottery amount 
- The Lottery will be conducted by an owner who would have access to start and end the lottery
- The owner will not have access to the funds
- The winner will be picked randomly based on the random number generated by an oracle
- Before starting to code, we must understand some basics first
- We will be covering
    - Randomness and its requirement in blockchain
    - Problem with generating on-chain randomness
    - About oracles and Random Number Generation
    - About Witnet Oracle
    - Random number generation contract from Witnet
## Requirement of Randomness
- Most applications require randomness in some sort
- Let's take for example a gambling contract(like our own), these contracts award people based on luck, and luck is random
- For these luck-based applications to work, the randomness must be tamper-proof so that no one can exploit
- While in the field of blockchain and smart contracts, there are some problems 
## Problem with On-Chain Randomness
- When people get started they use randomness by using the block number or the block timestamp
- But everything in the blockchain is deterministic
- This leads to big security concerns where a party can gain an unfair advantage
- The only bypass to this situation is to generate a number through a trusted source that is outside of the blockchain
- Enter Oracles!!
## Oracles and Random Number generation
- Oracles act as a bridge between smart contracts and the outer world
- Oracles are a reliable source of information
- They allow the blockchain to interact with external data
- Some of the data provided by oracles :
    - prices of commodities(for prediction markets)
    - weather conditions(for insurance contracts)
- One of the services that oracles provide is randomness generation
- Through oracles, smart contracts can obtain tamper-proof random numbers
- Every oracle has a different way of generating the randomness
- We are going to learn how to generate a random number using Witnet
## Witnet Oracle
- Witnet is a multichain oracle that gives smart contract access to real-world information
- It is one of the oracles that are available in the cell network
- Witnet provides us with a contract using which we can call and obtain randomness
- How this work is, nodes from the witnet are randomly selected to generate a random byte that is cryptographically committed
- It is called crowd-witnessing
- To learn more about randomness in witnet - https://docs.witnet.io/smart-contracts/witnet-randomness-oracle
## Using Randomness from witnet
- Before starting to code our lottery contract, let's understand the randomness functions provided by Witnet
- You can find their code and the explanation here - https://docs.witnet.io/smart-contracts/witnet-randomness-oracle/code-examples

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "witnet-solidity-bridge/contracts/interfaces/IWitnetRandomness.sol";

contract Witnet {

    uint32 public randomness;
    uint256 public latestRandomizingBlock;
    IWitnetRandomness immutable public witnet;
    
    constructor (IWitnetRandomness _witnetRandomness) {
        witnet = _witnetRandomness;
    }

    function requestRandomNumber() external payable {
        latestRandomizingBlock = block.number;
        uint _usedFunds = witnet.randomize{ value: msg.value }();
    }
    
    function fetchRandomNumber() external {
        assert(latestRandomizingBlock > 0);
        randomness = witnet.random(type(uint32).max, 0, latestRandomizingBlock);
    }
}
}
```
- Firstly we import the interface of the witnessRandomness contract
- The functionality is divided into two steps to be secure
- There is a request function and a fetch function
- First, we request a random number by paying a small fee
- it takes in the current block number and begins the process to generate a random number
- It takes around 5 to 10 minutes to generate the random number
- After 5 - 10 minutes, we can call the fetch function to obtain the random number
- This number is generated earlier and it is fetched into the contract whenever you call the fetch function
## Setup
- To make this tutorial as simple as possible, we are going to use only remix to write and test the contract
- You can also use local development by using your favorite code editor
- You can download the package for the interface of the randomness contract through this command:
    npm I witnet-solidity-bridge
- For the rest of us, we are going to use remix IDE
- Open up remix IDE and create a new file -> Lottery.sol
## Building the contract
- If you're curious, the entire code for the contract can be viewed by [CLICKING HERE!](https://github.com/KishoreVB70/Lottery-Tutorial/blob/main/Lottery.sol)
- Before starting to code a project, we must have an outline of all the functionality that will be in the contract
- This contract will behave like a traditional lottery where there is an owner who will start and end the lottery
- There will be people who will join the lottery by paying the lottery amount
- So the major functionality would be
    - Start lottery -> Only Owner
    - Join the lottery
    - End lottery(picking and awarding the winner) -> Only Owner
- In our case, we are going to incorporate the two-step random number generation from Witnet
    - Generate a Random number
    - Fetch Random Number
- Now as we have a rough idea about the functions, let's start the coding process
---
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "witnet-solidity-bridge/contracts/interfaces/IWitnetRandomness.sol";

contract Lottery {
    address witnetAddress = 0xbD804467270bCD832b4948242453CA66972860F5;
    IWitnetRandomness public witnet = IWitnetRandomness(witnetAddress);
```
- As usual, we write the license and the pragma solidity version
- We import the interface for the randomness contract
- We set the address of the randomness contract in Celo alfajores and create the instance of the interface.
---
```
    uint256 entryAmount;
    uint256 lastWinnerAmount;
    uint256 public lotteryId;
    uint256 public latestRandomizingBlock;

    address payable lastWinner;
    address[] players;
    address owner;

    bool open;
```
- Let's look at all the variables we will be going through
- Firstly we have the `entryAmount` so that the players can know the amount
- To give more information we have the last winner amount and the lottery Id
- We have the `latestRandomizingBlock` which is the block where we called the randomness function
- Coming to the address, we have:
    - Address of the owner of the contract
    - Address of the last winner(for information)
    - And finally an array of to track all the participants of the lottery
- Finally, we have a bool which shows if there is a current active lottery
---
```
    constructor () {
        owner = msg.sender;
    } 
  
    modifier onlyIfOpen{
        require(open, "Not Open");
        _;
    }

    modifier onlyOwner{
        require(msg.sender == owner, "not owner");
        _;
    }

    event Started(uint lotteryId, uint entryAmount);
    event Ended(uint lotteryId, uint winningAmount, address winner);
    error reEntry();
```
- We have a simple constructor where we set the creator of the contract as the owner
- We have specified two modifiers to control access
    - Firstly we have the `onlyOwner` modifier which allows only the owner to perform certain functionality
    - Then we have the `onlyIfOpen` modifier which allows access to certain functions only if there is a current active lottery
- Then we have two events to log the information about the lottery
    - `Started` event logs the ID of the event and the amount
    - `Ended` event logs the ID, the address of the winner, and the winner's amount
- We have also defined the custom error `reEntry` for users who try to enter the lottery more than one time
---
```
    function start(uint32 _entryAmount) external onlyOwner{
        //Check if there is a current active lottery
        require(!open, "running");
        open = true;
        entryAmount = _entryAmount * 1 ether;

        // Deleting the previous arrays of players
        delete players;
        emit Started(lotteryId, _entryAmount);
        lotteryId++;
    }
```
- First, we have the start function which can be called by the owner
- We have a require statement to prevent starting a new lottery when there is one currently active
- The function takes the entry amount and multiplies it to 1 ether unit to convert it to 1 celo
- For example: if the input is 10, then the fee would be 10 celo
- Then we update the state variables
    - First, we set open to be true as a new lottery is started
    - Then we clear the array of players that is left from the previous round
    - Then we update the `lotteryId`
- We also emit the `Started` event with the Id and the amount
---
```
    function join() external payable onlyIfOpen{
        require(msg.value == entryAmount, "Insufficient Funds");

        //Check if user is already a player
        for(uint i=0; i < players.length; i++){
            if (msg.sender == players[i]){
                revert reEntry();
            }
        }
        players.push(msg.sender);
    }
```
- Then we have the join function which is payable and can only be accessed if there is an active lottery(`onlyIfOpen`)
- First, we have a require statement to check if the user has sent the right amount
- Then we perform a check to see if the user is already entered in the lottery
- We have employed a simple for loop which iterated over the `players` array and checks if the caller is already a part of it
- If the caller is already in the array, then the function call is reverted by the custom error
- If not, the player is added to the array
---
```
    function requestRandomness() external onlyOwner onlyIfOpen{
        latestRandomizingBlock = block.number;

        uint feeValue = 1 ether;
        witnet.randomize{ value: feeValue }();
    }
```
- Next is the `requestRandomess` function which is a slightly modified version of the one from Witnet
- Instead of sending the funds from the caller,  we are going to use the funds already present in the contract to call the randomize function
- First, we set the block number
- Then we have a fee value which is set to 1 Celo(normal fee is very less than 1 Celo)
- This is not a problem as only the fee value will be deducted from the 1 Celo
- Finally, we call the randomize function in the randomness contract
---
```
    function pickWinner() external onlyOwner onlyIfOpen{
        assert(latestRandomizingBlock > 0);

        uint32 range = uint32(players.length);
        uint winnerIndex = witnet.random(range, 0, latestRandomizingBlock);

        lastWinner = payable(players[winnerIndex]);
        lastWinnerAmount = address(this).balance;

        (bool sent,) = lastWinner.call{value: lastWinnerAmount}("");
        require(sent, "Failed to send reward");

        open = false;
        latestRandomizingBlock = 0;
        emit Ended(lotteryId, lastWinnerAmount, lastWinner);
    }

    receive () external payable {}
```
- Finally, we have the `pickWinner` function which is going to end the lottery
- This function includes the fetching of the random number that has been generated
- This function must be called only after 5 - 10 minutes after the `requestRandomness` function to allow it to generate the random number
- Calling this function before that will result in reversion
- First, our function checks if the `requestRandomness` function has been called by checking the `latestRandomizingBlock` variable
- Then we close the lottery by setting `open` to false
- We also set the `latestRandomizingBlock` to 0 to prevent calling this function again
- Before fetching the random number, we are going to specify the range of the random number
- For our purpose, we have an array of addresses, and we need to select one person from that array
- So we specify the range to the length of the array(we would get a random number from 0 to range - 1)
- Then we call the random function as per the syntax provided by Witnet
- This will return us the random number which will be the index of the winner
- As the contract holds all the entry Amounts collected in the lottery, we are going to send the entire balance of the contract to the winner
- We update the global variables of the last winner's address and the winner's amount
- We use `call` to transfer the fund
- Finally, we emit the `Ended` event to log the information of the lottery Id, winner amount, and the winner's address
- As our contract is handling funds, we implement a receive function
---
## Testing the contract
- Finally, we are done coding the contract
- You can find the entire code of the contract by [CLICKING HERE!](https://github.com/KishoreVB70/Lottery-Tutorial/blob/main/Lottery.sol)
- Now we are going to use remix IDE and metamask to deploy the contract to the Celo alfajores network
- I have made a simple video to show you guys the functionality
[![Testing Video](https://i.postimg.cc/ZYFzPrvF/tutorial-yt.jpg)](https://www.youtube.com/watch?v=DPsizPEkZZk "Testing Video")
---
# Conclusion
- Congratulations!, you have learned another new implementation in the Web3 world
- In this tutorial, we have learned a reliable way  to generate random numbers and built a practical contract
---
# What's next
-  You can use this random number generator to make complex contracts for games
- Explore other oracles and use randomness generator from them
- You can learn more about oracles and access other forms of data they provide
---
# References
- Witnet - https://witnet.io/
- Witnet Randomness - https://docs.witnet.io/intro/tutorials/randomness
