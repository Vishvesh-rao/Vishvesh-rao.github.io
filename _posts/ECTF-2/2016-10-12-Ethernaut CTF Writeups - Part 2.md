---
title: Ethernaut CTF Writeups - Part 2
date:  2022-10-04 12:30:06
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

Continuing from where we left of......
___
## Level 7 [Force]

This challenge appraises on a certain unconventional way of sending ether to a contract.

To send ether as we normally do via *`transfer()`*, *`send()`* or *`call()`* to any contract we need it to be able to receive it via a function declared as payable.

But what if such a function is not present or in general are there any other ways to send ether to contracts otherthan the once mentioned above?... Turns out there are certain unconventional methods we can use to send ether to contracts or for that matter any valid ethereum address even if that address is not yet created or in use.

There are currently 3 ways to send ether to contracts/addreses even if they have no feature to receive ether:

- **Via block rewards:**
    If miner earns ether via sucessfull proof of work then the miner would receive a block reward and the miner can specify to which address this reward should be sent.
    
- **Sending to a Pre-calculated address:**
    As in the ethereum ecosystem calculating address is a deterministic process it is possible to easily calculate the address that would be assigned to a contract even before it is deployed thus if ether is send to that account beforehand than itwill have a non zro amount of eher on contract creation.
    
- **Self Destruct:**
    This is the most common one and includes calling the sucide opcode which will result in self destruct of entire contract code and its storageand all ether present in the contract will be sent to the specified address in the *`selfdestruct()`* function.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

As we can see the easiest way is to send ether to this contract via *`selfdestruc()`*.

All we need is a contract to which we need to send some ether and then call `selfdestruct(target_address)`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Exploit{
    constructor () public payable {
        selfdestruct(instanceAddress);
    }
}
```
___
## Level 8 [Vault]

This challenge enlightens us to a fundamental property of blockchain in general that is everything is publicly available unlike a normal codebase.

Everything from contract code timstamps to contracts storage is visible to everyone in the blockchain system. Hence care should be taken to never disclose sensitive information in smart contracts.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

So as we can see in the contract to pass this level we need to set the locked variable to false. It can be done by calling the unlock function but there is a check inside it to see if we provided the correct pasword only then will it set locked to false.

If we look at the declaration of password we notice it is set to private but then again this is all in the block chain and just because a variable is set to private doesent mean we cant read its value.

Private keyword only makes the variable safe from inheritance from other contracts and external contracts cant read the variable also unlike public variable which get a public getter function private variables dont have this.

But after all this when it comes to accessing the storage manually via a block explorer there is no difference in if the variable is public or private since all variables storage can be accessed easily.

So all we need to do is read the sorage slot where this variable is stored which in this case is slot 1.

```javascript
password = await web3.eth.getStorageAt(instance, 1)
```
___
## Level 9 [King]

This challenge brings to light the pull over push pattern thats should be followed whenever ether needs to be transferred.
It aims at shifting the risk associated with transferring ether to the user.

Sending ether to another address in Ethereum involves a call to the receiving entity. There are several reasons why this external call could fail. If the receiving address is a contract, it could have a fallback function implemented that simply throws an exception, once it gets called. Another reason for failure is running out of gas. Thus its always better to have the user call a withdraw() function rather than the contract transferring it itself.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

In this contract our goal is to block the contract from making anyone else king even if they send more ether than the current kings ether which is the check defined in the contract.

To this we need to just make sure the contract is never able to execute line 19 in whcih it assigns the new king. Also we notice that the contract is not following the pull over push pattern thus we can easily stop code execution when the contract transfers ether to a contract we specify.

All we need to do is declare a fallback fucntion in our contract which reverts eeverytime the contract reveives ether thus blocking the contracts code execution.

___

## Level 11 [Elevator]

This is just a starightforward challenge and more about just writing the correct contract code so that it passes the checks.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

So here the goal is to make the top variable to true. As we see the *`goto()`* fucntion is calling an external contract which implements a method called *`isLastFloor()`* and it takes one parameter.
On the first call to *`isLastfloor()`* it should return false to pass the check with the parameter `_floor` and once the check is cleared the same function should return true on calling it for a second time with the same parameter. 

The following is the code of the Building contract which achieves that:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Elevator {
    function goTo(uint _floor) external;
}

contract Building {
    bool private check = false;
    
    function isLastFloor(uint floor) external returns (bool) {
        
        if (check) {
            return true;
        } else {
            check = true;
            return false;
        }
    }
    
    function start(address elevatorInstance) public {
        Elevator elevator = Elevator(elevatorInstance);
        elevator.goTo(10);
    }
}
```

___

## Level 12 [Privacy]

This challenge familiarizes us with storage methods in solidity and how packing is done by the compiler to optimize storage space.

Each storage slot in solidity is of 32 bytes and storage occurs in a sequential way where the slot starts from 0 and variables can be grouped together if enough space is still available in a particular slot this is known as packing. Packing can be done only in case of statically-sized variable i.e complex types like mapping and dynamic sized arrays are always stored begining from a new slot and cannot be packed.

Below is the challenge contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

Now lets analyze the storage structure of the given contract.

```solidity
  bool public locked = true; // stored in slot 0 [1 byte]

  uint256 public ID = block.timestamp; // stored in slot 1 (complex type) [32 bytes]

  uint8 private flattening = 10; // stored in slot 2 [1 byte]

  uint8 private denomination = 255; // stored in slot 2 [1 byte] goes into slot 2 due to packing

  uint16 private awkwardness = uint16(now); // stored in slot 2 [2 bytes] goes into slot 2 due to packing

  bytes32[3] private data; // occupies slots 3,4,5 and 6 (complex type) [4x32 bytes] as complex type it starts from a new slot
```

Now on line 17 we can see that function *`unlock()`* takes in a key as a parameter and that key is checked with the first 16 bytes of the value stored in the 3rd storage variable of the `data` array.

Looking at the analysis of the storage pattern of this contract we see that `data[2]` will be stored in the 5th slot so all we need to do is read that memory slot and take the first 16 bytes and pass it to the *`unlock()`* function to pass this level.

___

## Level 13 [Gatekeeper One]

As the name suggests this challenges consists of 3 gates which we need to pass in order to solve it. Each gate is a require statement enclosed in a modifier.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### gateOne
```solidity
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```
This is the first gate and as we can see the check is similar to the Telephone challenge we did where we use another contract to act as intermediary.

### gateTwo
```solidity
  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }
```
This statement checks if the gas left modulo 8191 is equal to 0 i.e if 8191 divides the gas left this can achieved by simple brute force passing different values of gas until the check passes.

### gateThree
```solidity
  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }
```

This modifer has 3 seperate checks all based on conversion between different uint types:

The way uint conversion works is when you convert a uint from higher uint to a lower one the first x bits of the original uint value will be taken when converting from uint to uintx i.e when converting uint64 to uint32 the first 32 bits of the vaiable will be taken.

1. check one:
```solidity
  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
```
this checks matches the lower 32 bits with lower 16 bits of `_gatekey` to get them to be equal we can make bits $16-31$ of `_gateKey` as 0.

2. check two:
```solidity
require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
```
Here the lower 32 bits is checked with the 64 bits of `_gateKey` and as they should not be equal i.e the upper 32 bits of `_gateKey` should not be 0. We see in a way this is kinda the opposite of the first check.

3. check three:
```solidity
quire(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
```
This check is easy enough to pass all we need to do is ensure when we make the `_gatKey` its lower 16 bits are equal to lower 16 bits of the transaction address.

in the end consolidating all these we need `_gateKey` which is 64 bits whose

- first 16 bits are equal to that  of tx.origin
- next 16 bits are 0
- next 32 bits shouldnot be 0

___

## Level 14 [GateKeeper Two]

Similar to the above challenge this also requires us to pass 3 gates.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### gate One

```solidity
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```

This is similar to last challenge and the Telephone challenge.

### gate Two
```solidity
  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }
```
This gate checks if the address calling the function on which this modifier is used is smart contract or not. The way it does this is by making use of the extcodesize opcode which returns the length of the code present if any given an address. thus if smart contract is present at that address it  will return a number greater than 0 and the check will fail. But since to pass the first check we will need to use a smartcontract.

To bypass this we can call this contract from the constructor of our custom made smart contract. What this will do is fool the extcodesize opcode as during constrctor intilization the runtime bytecode has not yet been generated and hence the compiler doesnt know the size of the code present in the contract thus it will be 0 at this time. Hence allowing us to pass this check.

### gate Three

```solidity
 modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }
```

This gate checks if the `_gateKey` is correct via a series of xor operations and at the end the xored result should satisfy a condition.
But since we know all the variables other than `_gateKey` in this equation it is easy to recover it and pass the final check.
___