---
title: # STABLEMAGNET SWAP: ANALYSIS OF A RUGPULL
date:  2023-01-10 12:30:06
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


Around $22.2 (currently 27 and increasing) BUSD was rug pulled from StableMagnet Swaps liquidity pool on 23rd June 2021. This is an analysis of the exploit....

## Addresses

- Rug Pull Transaction : [0xf0ba46c8a20e1e75ad382e42c509bf71393e1b3b90326165c747a5d3cc5d967c](https://bscscan.com/tx/0xf0ba46c8a20e1e75ad382e42c509bf71393e1b3b90326165c747a5d3cc5d967c)
- StableMagnet Swap Contract: [0xb89e9365cb5bacfcf4a4b0386dfad45c3b4d3258](https://bscscan.com/address/0xb89e9365cb5bacfcf4a4b0386dfad45c3b4d3258)

- SwapUtils Library: [0xE25d05777BB4bD0FD0Ca1297C434e612803eaA9a](https://bscscan.com/address/0xE25d05777BB4bD0FD0Ca1297C434e612803eaA9a)
- BUSD sent to BNC: [Txn's](https://bscscan.com/address/0x2bac04457e5de654cf1600b803e714c2c3fb96d7#tokentxns)
- Tether on EC: [Txn's](https://etherscan.io/address/0xDF5B180c0734fC448BE30B7FF2c5bFc262bDEF26#tokentxns)
- Tether to DAI: [Txn's](https://etherscan.io/address/0xe5daac909a3205f99d370bc2b32b1810a4912a07#tokentxns)


## Overview

The protocol orgs managed to rugpull 22.2 million dollars

[This](https://bscscan.com/tx/0xf0ba46c8a20e1e75ad382e42c509bf71393e1b3b90326165c747a5d3cc5d967c) is the tranasction that the intiated the rugpull which included 3 token transfres totaling to 22.2 million dollars worth.


Following thw witdrawal all funds were split and transfereed to multiple wallets and converted to tether and finally converted to DAI


## Transfer Of Funds
The following is the rough steps of the transactions that occurred:

[![](https://mermaid.ink/img/pako:eNptk1tv2jAUx7-KlfFIke-xeZgEhPSyVZsG1aRBH44dp0QNCQpGpQ189xlIp93ylHPx7__PyXEb2Tpz0TB6amCzQvNkWaHwjBYj78E-u2aLvkNZOv-Irq4-HvBeGQdaZ5xwq63AxhkupGVWx8QpzaQhgljD9AF1qPFi9gKbB1-UWzSpK9-A_QWbUpFhEcfxeMxNgtMET4BQHU84404SqjBzMNJwQJMLbLKYeTClu4enynl0Iv8DNUq7YERYIwzY3OYcuMFMySyHjAvLDM8YFSo4bL_tntDXXVmieQPVNkCKukLLoNWjdEDRPRo_zJLjRTvp-Dk2wKVVQLEjLhaQMUUdp2Ea2uQxYZo5YpjRmFFJgo-YxyAyZq3ItIztAU3bv6d71txuysIjv3Io31XZ9pzrtKdn7ZdL75fKHVC66O0ffy92oPlLfUDXi97rf4urxoWzN4veW1dOT2V0246LCirr0E3tO0ud8vWlAV2imz-i224iNMwZcy5iJzInBbc5kRib09-Lw55Qy3KjZRYf0F07Dd_XuN0aTVZQVJ3IXQdKUjEmClscM55POFfjKcPjOE3DcE1qqaQmmaZUfvD1s6v8vtoe0Kd27k7M91F96ljBC4DVWAOjWORhZ1mMjaWGUUMUwcA1oYCDqc9tMroNp6N-tHbNGoos3If2RFtGgbx2y2gYXjOXw670y2hZHUPrbpOBd9Os8HUTDX2zc_0Idr6evVb2Pb70JAWE27WOhjmU25DdQPWjrtfvTSGMhm20j4ZU0YFgWFOphaaEcNmPXkOaDcL26nBPGJFSEcaO_ejtTMADrSQTgilFNCNC0eNPhPIhMg?type=png)](https://mermaid.live/edit#pako:eNptk1tv2jAUx7-KlfFIke-xeZgEhPSyVZsG1aRBH44dp0QNCQpGpQ189xlIp93ylHPx7__PyXEb2Tpz0TB6amCzQvNkWaHwjBYj78E-u2aLvkNZOv-Irq4-HvBeGQdaZ5xwq63AxhkupGVWx8QpzaQhgljD9AF1qPFi9gKbB1-UWzSpK9-A_QWbUpFhEcfxeMxNgtMET4BQHU84404SqjBzMNJwQJMLbLKYeTClu4enynl0Iv8DNUq7YERYIwzY3OYcuMFMySyHjAvLDM8YFSo4bL_tntDXXVmieQPVNkCKukLLoNWjdEDRPRo_zJLjRTvp-Dk2wKVVQLEjLhaQMUUdp2Ea2uQxYZo5YpjRmFFJgo-YxyAyZq3ItIztAU3bv6d71txuysIjv3Io31XZ9pzrtKdn7ZdL75fKHVC66O0ffy92oPlLfUDXi97rf4urxoWzN4veW1dOT2V0246LCirr0E3tO0ud8vWlAV2imz-i224iNMwZcy5iJzInBbc5kRib09-Lw55Qy3KjZRYf0F07Dd_XuN0aTVZQVJ3IXQdKUjEmClscM55POFfjKcPjOE3DcE1qqaQmmaZUfvD1s6v8vtoe0Kd27k7M91F96ljBC4DVWAOjWORhZ1mMjaWGUUMUwcA1oYCDqc9tMroNp6N-tHbNGoos3If2RFtGgbx2y2gYXjOXw670y2hZHUPrbpOBd9Os8HUTDX2zc_0Idr6evVb2Pb70JAWE27WOhjmU25DdQPWjrtfvTSGMhm20j4ZU0YFgWFOphaaEcNmPXkOaDcL26nBPGJFSEcaO_ejtTMADrSQTgilFNCNC0eNPhPIhMg)

## Exploit Analysis


![](https://i.imgur.com/DxifpXB.png)

SO as we can see the main exploit occured in an external call to the StableMagnet contract.

This external call was to a library called [SwapUtils](https://bscscan.com/address/0xE25d05777BB4bD0FD0Ca1297C434e612803eaA9a) which had a method which enabled the orgs to transfer all the liquidity to their own wallet.

The attackers took advantage of the fact that the block explorers only verify the given contracts source code and not of the libraries which it may be importing this enabled them to have hidden functions as the code for this contract was not public.

Not only did it have method to enable transfer of funds it also had the utility to keep extracting even more funds from the wallets of those users which had approved StableMagnet to deduct the set allownace in return for more tokens.

## Prevention

This rug pull highlights certain flaws..

One we should never trust the verified check mark, it just verifies the code supplied to it and even if there is no use of external libraries in the code, verification is no guarentee towrds the safety of the code in the contract.

This rug pull was an example of a hidden transfer which is not visible to the public but when executed transfers all the liquidity to the owners contract/wallet.

Again this calls for stronger awareness on the part of investors especially in conducting at least a brief audit of the token into which they are investing or at least going thorugh the audit performed by a third party and to check if it seems legitimate and if all the issues mentioned have been confirmed with proof by the orgs that they have been fixed.


