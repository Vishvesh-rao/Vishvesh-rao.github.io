---
title: Ethernaut CTF Writeups - Part 1
date:  2022-11-04 12:30:06
author: Vishvesh
author_url: https://twitter.com/The_Str1d3r
mathjax: true
categories:
  - Blockchain
  - CTF-Writeups
tags:
  - "Ethereum"
  - "Solidity"

url : https://hackmd.io/Cyo2eYrKQPSvN3Zp8pga9g?view
mermaid: true
---
<html>
  <head>
    <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
    </script>


<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS_HTML">
  MathJax.Hub.Config({
    "HTML-CSS": {
      availableFonts: ["TeX"],
    },
    tex2jax: {
      inlineMath: [['$','$'],["\\(","\\)"]]},
      displayMath: [ ['$$','$$'], ['\[','\]'] ],
    TeX: {
      extensions: ["AMSmath.js", "AMSsymbols.js", "color.js"],
      equationNumbers: {
        autoNumber: "AMS"
      }
    },
    showProcessingMessages: false,
    messageStyle: "none",
    imageFont: null,
    "AssistiveMML": { disabled: true }
  });
</script>
</head>
</html>

------

I recently started attempting Ethernaut CTF challenges and it has been a great learning experience solving all these variety of challenges which cover a broad spectrum of topics in solidity security and best practices and some coding tricks, gas reduction techniques etc. Every challenge brings forward a new topic and lot of the levels are connected too making it more easier to connect different bugs and their solutions.

So this series of blog posts is just a way for me to document my findings for each challenge so that I can refer back to them if/when I forget in the future how I solved them or just want to look back at my methods at a later date.

So lets dive into it.....
___

## Level 0 [Hello World]

This is the introductory level where the entire level consistis of calling certain methods each pointing to another method name and at the end we need to authenticate with a password giving the password as a parameter to the `authenticate()` method. This level basically teaches us to use the console which is what we use to get the password which can be viewed through the password method. We get knowledge of the password method by viewing the list of contract objects via the console. The password comes out to be *`"ethernaut0"`*

Below is the list of commands we need to enter into the console to complete the level:

```javascript
await contract.info()

await contract.info1()

await contract.info2()

await contract.infonum()

await contract.info42()

await contract.theMethodname()

await contract.method7123949()

await contract.password()

await contract.authenticate("ethernaut0")
```

___
## Level 1 [Fallback]

This level as the name describes familiarizes us with the *fallback* function in solidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```
As we can see there is a function called receive which when called sets the callers address as the owner of the contract. So our goal is to somehow make the contract execute this recieve function. Fallback functions in solidity are called when a contract gets called with ether passed along with the call and additionally some data may or may not be sent along. If the data field is not empty then the *`fallback()`* function is called else the *`recive()`* fucntion is called.

Thus we just send over a message call to the contract keeping the data field empty thus resulting in the execution of the recieve function and making us the owner of the contract. Therafter its straight forward to drain the contract of all the ether calling the *`withdraw()`* function.

Below is the code to execute the transaction calling the contract:

```javascript
await contract.contribute({value:toWei(0.001)})
```
___

## Level 2 [Fallout]

This level is an intro to what constructors are in a way.
Constructors in solidity are functions which are called on contract intilization before the rest of the code is loaded. In solidity broadly there are two types of bytecodes the creation and runtime bytecode.

The constructor logic and its corresponding parameters etc get intialized in the creation bytecode that is generated on deployment of the smartcontract. It includes the input data to the transaction that creates the contract.

The creation bytecode intializes all the variables declared inside the constructor and sebsequently genrated the runtime code which includes the rest of the smart contract logic other than the constrctor.

This level highlights a classic flaw in the way constructors were allowed to be declared from solidity version `^0.6.0`. Constructors were basically named after the contract name itself like so:

```solidity
contract example {
    function example(){
        ...
    }
}
```
now keeping this we will lokk at the challenge code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

Here we can clearly see that the constructor is named improperly the contract name is Fallout while the constructor is named as Fal1out.

This will result in being not considered as a fucntion on compilation.
rather this fucntion will be considered as anormal function and this means after deployment anybody could call this function which would not have been the case in the case of a constructor.

Thus we just need to call this method and as it sets the msg.sender as the owner we will be set as the owner of the function of the contract.

Below is the code:

```javascript
await contract.fal1out()
```

___
## Level 3 [Coin Flip]

This challenge deals with the concept of randomness in blockchain. In blockchain since everything is deterministic there is no true source of randomness unless we bring some external oracle into the picture to provide a source of randomness.

In the challenge as we can see below its basically about guessing the value of a boolean variable correctly 10 times and on guessing wrong even once the the variable tracking the correct guesses is reset to 0.

```solidity=
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

As we can see on line 18 the randomness is generated from the hash of block number.

```solidity
uint256 blockValue = uint256(blockhash(block.number.sub(1)));
```

From line 26 we see that we need the value of the side variable to get a correct guess and side is calculated from blockValue which is derived from subtracting 1 from the current block number.

blockValue is further divided by FACTOR of which we know the value of and its result stores in coinFlip.

Hence what you migh notice here is that everything in this is already either known to us or can be easilt determined. This means we can just make the flip function ourself to correctly guess the value of side variable and submit it after calling the flip funcion doing these 2 calls 10 times will get us past the level.


___
## Level 4 [Telephone]

This level illustrates a classic phishing method that can be used to gain access of a smart contract that make use of such authenticating pattern.

The contract uses `tx.origin` check to assign the owner address, the problem is `tx.origin` is vulnerable to phishing attacks which can trick users into performing authenticated actions on the vulnerable contract.

```solidity=
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

The goal of this challenge is to pass the check on line 13 and become the owner. If we call the *`changeOwner()`* function directly with our player address then the `tx.origin` which is the address drom which the transaction originated (in this case our player address) and `msg.sender` (which will also be player address as we called the function) will be same and the check will fail.

To bypas this we make use of another contract say attack contract which will call the above contract call this the target contract. As calling another contract also is a message call hence when we call the attack contract the `tx.orgin` will be set to our address (player address) and this attack contract calls function in target contract making the `msg.sender` address as the attack contracts address as it is the one calling the *`changeOwner()`* function in target contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Telephone {
  function changeOwner(address _owner) external;
}

contract Attack {
    Telephone telephone;
    
    constructor(address telephoneInstance) {
        telephone = Telephone(telephoneInstance);
    }
    
    function changeOwner() public {
        telephone.changeOwner(msg.sender);
    }
}
```

___
## Level 5 [Token]

This challenge is based on  arithmetic overflow/underflow in solidity. 

So lets understand what exactly this concept is first.

For all the data types present in Solidity the EVM has a specified size fixed for that particular data type. Meaning a certain data type can only represent values in certain range.

this concept becomes especiall intresting when it comes to integers as a basic uint8 can represent numbers from $0$ to $2^8-1$ i.e [0,255] this is the range of an 8 bit unsigned integer variable.

So integer overflow occurs when an unsigned integer variables value exceeds the range specified for that type and it cant hold that big a value anymore, in which case the value of that variable will be reset to its minimum value i.e 0 and if any excess is there then it will increase from 0 herafter i.e $255 + 1$ will result in $0$ while $255 + 2$ will result in $1$.

That was overflow in the case of integer underflow the same occurs just in reverse i.e subtracting 1 from 0 will result in 255 and so on, basically wrapping around but in reverse this time.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

As we can see we are provided with 20 tokens intially and to win we need to increase the number of tokens to a large value somehow.

The transfer function takes in 2 parameters, the amt to transfer and the adress to which to transfer it to. It checks if the sender has that amount to transfer at first then it subtracts that amount from the senders balance and adds it to the recivers balance.

The issue here is where it deducts the amount frm the senders balance:

```solidity!
balances[msg.sender] -= _value;
```

This line has the ability to underflow reason being there is no checks on the subtraction.
Checks for overflow/underflow were not inherantly implemented until solidity version 0.8.0 and above.

Until then the developer had to make use of the safeMath library importing it explicitly and using it everytime an addition/subtraction needed to be performed and in this case looking at the solidity version we see that it is below 0.8.0 menaing it is vulnerable to underflow/overflows and since safeMath has not been called it is definetly going to cause underflows and overflows.

Taking advantage of this we can send to the transfer function some random address to which we want to transfer tokens and as value for the amount to transfer we can set it to 21.

This will result in the subtraction of 20 - 21 when the balance is being deducted from our account and this subsequently will result in an underflow and the value of our balance i.e the senders balance will be reset to the highest value that can be held by a uint variable which will result in a huge amount of tokens in our contract. 

After this we simply need to submit our instance and we will have passed this level.
___
## Level 6 [Delegation]

This level illustrates the useage of delegate call method in solidity and why it should be used with the utmost precaution and how to exploit a delegate call not used properly.

Delegate call opcode executes the code in the called contract in the context of the calling contract i.e things like storage and msg.sender and msg.value remain unchanged.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

Storage in solidity can be understood as certain slots with each variable being assigned a slot starting from 0.

In `Delegation` contract in slot 0 the variable `owner` is stored in slot 0 and in `Delegate` contract too the variable `owner` is present in slot 0 and in `Delegate` contract we are modifying this `owner` variable inside *`pwn()`* function, this variable is present in slot 0 of `Delegate` contract, i.e the variable present in slot 0 of `Delegation` contract will be modified on calling *`pwn()`* function as the call is a delegate call which modifies the corresponding storage slot in the calling contract.

Thus if we call via delegate call the *`pwn()`* function in Delegate contract with player address our address will be set as the owner of the Delegation contract and thereby we can pass the challenge.

To call *`pwn()`* function we need to encode the function's signature in the `msg.data` field. In solidity encoded function signature of any function is the starting 4 bytes of the $keccak256$ hash of the functions signature. Function signature comprises of the function name along with the data type of parameters seperated by commas and enclosed in paranthesis.

The following code can be input into the console to pass the challenge:

```javascript
await sendTransaction({from: player, to: instance, data: "0xdd365b8b"})
```
___
