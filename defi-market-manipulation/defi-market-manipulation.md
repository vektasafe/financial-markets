# How DeFi Protocols Get Exploited Through Market Manipulation

**Author:** Vektasafe
**Category:** Financial Markets Research
**Topic:** Market manipulation attack vectors in decentralised finance -- mechanics, documented incidents, and mitigations

---

## Overview

Market manipulation is the deliberate distortion of asset prices or trading conditions to extract value from other participants. In traditional finance, it is illegal, heavily monitored, and prosecuted by regulators including the SEC, CFTC, and FCA. In decentralised finance, the same behaviours occur -- often with greater sophistication -- in an environment where enforcement is limited, pseudonymity is common, and the protocols themselves are the counterparty.

DeFi market manipulation is not a fringe concern. It is one of the primary mechanisms through which value is extracted from protocols and their users. Between 2020 and 2024, manipulation-related exploits accounted for billions in losses across hundreds of protocols. Unlike smart contract bugs, which require finding a code vulnerability, manipulation attacks often require only capital, timing, and an understanding of how a protocol's incentive structure creates exploitable dynamics.

This document covers the primary manipulation attack classes used against DeFi protocols -- how they work mechanically, which protocols have been targeted, what the losses were, and what mitigations exist. The goal is to give the reader a practitioner-level understanding of manipulation as a security category, not just a financial one.

---

## 1. The DeFi Market Structure and Why It Is Vulnerable

### 1.1 How DeFi Markets Differ From Traditional Markets

Traditional financial markets have several properties that make manipulation difficult:

**Centralised order books** -- All trades on an exchange pass through a single matching engine. The exchange can monitor for unusual patterns, freeze accounts, and report suspicious activity to regulators.

**Identity requirements** -- Participants must pass KYC/AML checks to trade. This creates accountability and allows regulators to trace manipulative behaviour to individuals.

**Circuit breakers** -- Exchanges implement automatic trading halts when prices move beyond set thresholds within a time window. This limits the speed and magnitude of manipulation.

**Market surveillance** -- Dedicated teams and automated systems monitor for wash trading, spoofing, layering, and other manipulation patterns in real time.

DeFi has none of these by default. Trades execute against smart contracts that cannot distinguish between legitimate trading and manipulation. Participants are pseudonymous. There are no circuit breakers unless explicitly programmed. Market surveillance is entirely reactive -- analysing on-chain data after the fact.

### 1.2 The Role of Liquidity Pools

Most DeFi trading occurs through automated market makers (AMMs) rather than order books. An AMM like Uniswap holds reserves of two tokens and prices them according to a mathematical formula -- typically x * y = k, where x and y are the token reserves and k is a constant.

This pricing model has a critical property: the price is determined entirely by the ratio of reserves. Anyone with enough capital can move the price by trading against the pool. This is not manipulation in itself -- it is how the protocol works. But it creates a mechanism that can be weaponised.

### 1.3 The Role of Oracles

Protocols that need external price data -- lending markets, synthetic asset protocols, perpetual exchanges -- rely on oracles to bring that data on-chain. An oracle is a system that reads a price from somewhere (a DEX, a centralised exchange, an aggregator) and writes it to the blockchain where smart contracts can read it.

The oracle is the interface between the real world and the protocol. If the oracle reads from a source that can be manipulated, the protocol can be fed false prices. If the protocol makes financial decisions -- liquidating positions, settling trades, minting synthetic assets -- based on those false prices, value can be extracted.

These two elements -- AMM pricing mechanics and oracle dependencies -- are the foundation of most DeFi market manipulation attacks.

---

## 2. Flash Loan Attacks

### 2.1 What They Are

A flash loan is an uncollateralised loan that must be borrowed and repaid within a single blockchain transaction. If the repayment condition is not met, the entire transaction reverts -- the loan never happened. This atomic guarantee is what makes flash loans safe for lenders.

For attackers, flash loans provide instant access to enormous amounts of capital with no upfront cost and no credit risk. An attacker with no capital of their own can borrow tens or hundreds of millions of dollars, execute a sequence of operations across multiple protocols, and repay the loan -- all within a single transaction that takes approximately 12 seconds on Ethereum.

Flash loans did not create manipulation attacks. They made them dramatically cheaper and more accessible by removing the capital requirement.

### 2.2 How They Are Used in Manipulation

The typical flash loan manipulation attack follows this sequence:

1. Borrow a large amount of a token via flash loan
2. Use the borrowed tokens to manipulate a price on a DEX
3. Exploit a protocol that reads from that DEX as a price oracle
4. Reverse the price manipulation
5. Repay the flash loan
6. Keep the profit

The entire sequence executes atomically. If any step fails, the whole transaction reverts. If it succeeds, the attacker walks away with the profit from step 3.

### 2.3 Real Incident -- bZx Protocol (2020)

**Date:** February 2020
**Loss:** 954,000 USD across two attacks

The bZx attacks were the first high-profile flash loan manipulation incidents and established the template for dozens of subsequent attacks.

**Attack 1 (February 14, 2020):**

The attacker borrowed 10,000 ETH via a flash loan from dYdX. They used 5,500 ETH to open a short position on WBTC/ETH on bZx. They then borrowed 112 WBTC from Compound using 5,500 ETH as collateral, and dumped the WBTC on Uniswap, crashing the WBTC price on that pool. bZx used Uniswap as its price oracle. The protocol saw the WBTC price had fallen and settled the attacker's short position at a profit. The attacker repaid the flash loan and kept approximately 360,000 USD in profit. The entire attack executed in a single transaction.

**Attack 2 (February 18, 2020):**

Four days later, the attacker borrowed 7,500 ETH via flash loan, manipulated the sUSD price on Kyber Network, then used the inflated sUSD to borrow ETH from bZx at a favourable rate, draining approximately 600,000 USD.

**Why It Worked:**

bZx used single-source price oracles. A single source with sufficient liquidity can be temporarily moved by a flash-loan-sized trade. The protocol had no mechanism to detect or reject prices that had moved dramatically within the same transaction.

### 2.4 Real Incident -- Pancake Bunny (2021)

**Date:** May 2021
**Loss:** 45,000,000 USD

Pancake Bunny was a yield optimiser on Binance Smart Chain. The attacker borrowed a massive amount of BNB via flash loan, used it to buy BUNNY tokens on PancakeSwap, dramatically inflating the BUNNY price, and then triggered Pancake Bunny's price-based minting function -- which minted new BUNNY tokens based on the inflated price. The attacker received an enormous amount of newly minted BUNNY tokens, immediately dumped them, repaid the flash loan, and extracted approximately 45 million USD. The BUNNY token price collapsed by over 95% within minutes.

**Why It Worked:**

The protocol's minting mechanism used the spot price from PancakeSwap without any time-weighted averaging or deviation check.

### 2.5 Mitigations

- Never use spot price from a single DEX as a protocol oracle -- use TWAP which cannot be manipulated within a single transaction
- Use decentralised oracle networks such as Chainlink that aggregate from multiple sources
- Implement price deviation checks that pause protocol actions when prices move beyond expected ranges
- Separate execution and settlement with a time delay, making same-block manipulation impossible

---

## 3. Oracle Manipulation Without Flash Loans

### 3.1 What It Is

Not all oracle manipulation requires flash loans. An attacker with sufficient capital can manipulate oracle prices over longer time periods -- minutes, hours, or days -- to position themselves advantageously before triggering a protocol action.

This category of attack is more patient, more capital-intensive, and harder to detect in real time. It is also more common than flash loan attacks against more sophisticated protocols that have already mitigated single-block manipulation.

### 3.2 Real Incident -- Mango Markets (2022)

**Date:** October 2022
**Loss:** 114,000,000 USD

Mango Markets was a decentralised trading platform on Solana. The attacker, later identified as Avraham Eisenberg, executed a multi-step manipulation of the MNGO token price.

**Attack Sequence:**

1. The attacker created two accounts on Mango Markets
2. From Account A, they sold a large amount of MNGO perpetual futures contracts
3. From Account B, they bought spot MNGO on multiple exchanges simultaneously, driving the price from approximately 0.03 USD to 0.91 USD
4. Mango Markets used the spot price of MNGO to calculate collateral value
5. At the inflated price, Account A showed enormous unrealised profits
6. The attacker used these phantom profits as collateral to borrow 114 million USD from the Mango treasury
7. The borrowed assets were withdrawn before the price collapsed back to its real value
8. The collateral was insufficient to repay the loans -- the protocol was left insolvent

Eisenberg was later arrested by US authorities and charged with commodities fraud and market manipulation -- one of the first criminal prosecutions arising from a DeFi exploit.

**Why It Worked:**

Mango Markets used a single price feed for MNGO that could be moved by spot market buying. The protocol allowed inflated collateral value to be used immediately for borrowing without any time delay or deviation check.

### 3.3 Real Incident -- Cream Finance (2021)

**Date:** October 2021
**Loss:** 130,000,000 USD

The attacker used a flash loan of 500 million DAI to manipulate the price of yUSD which Cream used as collateral. The attacker deposited yUSD at the inflated price, borrowed against it, and extracted 130 million USD in various assets.

### 3.4 Mitigations

- Collateral value caps -- limit total borrowing power of any single collateral asset regardless of reported price
- Borrow rate limiting -- limit how much can be borrowed against any collateral position within a time window
- Conservative collateral factor -- require significantly more collateral than borrowed value to buffer against price manipulation
- On-chain manipulation detection -- monitor for large rapid price movements and pause borrowing when detected

---

## 4. Sandwich Attacks

### 4.1 What They Are

A sandwich attack is a form of front-running specific to AMM-based trading. When a user submits a large trade to an AMM, that trade is visible in the mempool before it is confirmed on-chain. An attacker -- typically a bot -- inserts a buy before the victim's trade and a sell after it, extracting the price impact caused by the victim's trade.

### 4.2 How It Works Mechanically

Consider a user trading 1,000,000 USDC for ETH on Uniswap. The attacker's bot detects this pending transaction and submits two transactions with higher gas fees:

- **Front-run:** Buy ETH with USDC before the victim's trade, slightly increasing the ETH price
- **Back-run:** Sell ETH for USDC after the victim's trade executes at the now-higher price

The victim's 1,000,000 USDC buys less ETH than it should have. The difference goes to the attacker.

### 4.3 Scale of the Problem

Sandwich attacks are not occasional events. They are systematic, automated, and ongoing. Research by Flashbots and independent analysts estimated that over 1.38 billion USD was extracted from DeFi users through sandwich attacks between 2020 and 2023. MEV bots run continuously scanning the mempool for profitable opportunities. Individual victims typically lose 0.1% to 2% of their trade value -- small per transaction, enormous in aggregate.

### 4.4 Mitigations

- Tight slippage tolerance -- set maximum slippage of 0.1% to 0.5% so sandwich attacks cause the transaction to revert
- Private transaction pools -- use Flashbots Protect or MEV Blocker to submit transactions directly to block builders, bypassing the public mempool
- Commit-reveal schemes -- require traders to commit to a trade before revealing details, preventing front-running
- Batch auctions -- aggregate trades over a time window and settle at a single clearing price, eliminating individual front-running (used by CoW Protocol)

---

## 5. Wash Trading and Artificial Volume

### 5.1 What It Is

Wash trading is buying and selling the same asset between accounts you control to create the appearance of trading activity. In DeFi, it is used to farm volume-based protocol rewards, create false price signals, manipulate platform rankings, and qualify for airdrops.

### 5.2 Real Incident -- LooksRare NFT Platform (2021-2022)

**Date:** 2021-2022
**Scale:** Billions in artificial volume

LooksRare distributed LOOKS token rewards to traders based on trading volume. Wash traders generated billions in artificial volume to farm LOOKS tokens. At peak, wash trading accounted for an estimated 95% of LooksRare's reported volume. Research by Chainalysis confirmed that a small number of wallets were responsible for the vast majority of this activity.

### 5.3 Real Incident -- DEX Volume Inflation

Multiple DEX protocols distributing liquidity mining rewards based on trading volume metrics saw systematic wash trading follow. Research on early Uniswap competitors found that a significant proportion of reported volume on incentivised pairs was circular -- the same wallets buying and selling to themselves to earn protocol tokens worth more than the gas costs of the trades.

### 5.4 Mitigations

- Fee-based volume measurement -- trading fees make wash trading expensive and limit its profitability
- On-chain wash trading detection -- analyse wallet graphs to identify circular trading patterns
- Time-weighted activity metrics -- reward sustained activity over time rather than raw volume
- Sybil-resistant identity -- use on-chain reputation systems to limit reward farming by coordinated wallet clusters

---

## 6. Governance Manipulation

### 6.1 What It Is

Most DeFi protocols are governed by token holders. Governance tokens give holders the right to propose and vote on protocol changes. This creates a new attack surface: acquiring enough governance tokens to pass malicious proposals. Governance manipulation does not require exploiting a code vulnerability -- it requires acquiring voting power and using it to transfer value to the attacker.

### 6.2 Real Incident -- Beanstalk Farms (2022)

**Date:** April 2022
**Loss:** 182,000,000 USD

Beanstalk was a credit-based stablecoin protocol. Its governance allowed anyone to propose and vote on changes, with voting power proportional to Stalk token holdings. Stalk could be acquired and used to vote within the same transaction.

**Attack Sequence:**

1. The attacker took out a flash loan of approximately 1 billion USD in various assets
2. Used the borrowed funds to acquire approximately 79% of voting power
3. Voted to pass a malicious governance proposal submitted 24 hours earlier disguised as a charitable donation
4. The proposal transferred all protocol assets to the attacker's wallet
5. The attacker repaid the flash loan and kept 182 million USD

**Why It Worked:**

Beanstalk's governance had no time delay between proposal submission and execution, and no minimum holding period for voting power. Flash loans provided instant supermajority control with no capital commitment.

### 6.3 Real Incident -- Build Finance (2022)

**Date:** February 2022
**Loss:** 470,000 USD (entire treasury)

A single attacker accumulated enough BUILD governance tokens over time to pass a proposal giving themselves minting rights over the protocol's treasury token. They then minted and sold the entire treasury.

### 6.4 Mitigations

- Timelock on governance execution -- require a mandatory delay of 48-72 hours minimum between a proposal passing and executing
- Voting power snapshots -- take a snapshot of token holdings at proposal submission time so flash-loan-acquired tokens cannot vote
- Quorum requirements -- require a minimum percentage of total token supply to participate in a vote
- Guardian multisig -- maintain an emergency veto that can cancel malicious proposals during the timelock period

---

## 7. Coordinated Liquidation Attacks

### 7.1 What It Is

Lending protocols liquidate undercollateralised positions when collateral value falls below a threshold. Under manipulation, an attacker can deliberately trigger liquidations by crashing a collateral asset's price, then front-run the liquidation process to capture the liquidation bonus at the expense of borrowers.

### 7.2 Real Incident -- Compound DAI Liquidations (2020)

**Date:** November 2020
**Loss:** 89,000,000 USD in liquidated positions

The DAI price on Coinbase Pro -- which Compound used as its price oracle -- spiked from approximately 1.00 USD to 1.30 USD due to a surge in demand and low liquidity on that specific exchange. Compound triggered liquidations on DAI-collateralised positions that were actually healthy at the real market price. Liquidation bots immediately triggered on the false signal. Approximately 89 million USD worth of positions were liquidated at a significant discount to their real collateral value.

### 7.3 Real Incident -- Venus Protocol (2021)

**Date:** May 2021
**Loss:** 100,000,000 USD

The XVS token price was manipulated upward on Binance Smart Chain. Large positions had been opened on Venus Protocol using XVS as collateral. At the inflated price, the attacker borrowed heavily against the XVS collateral, then allowed the price to collapse. The borrowed funds were extracted, the XVS collateral became worthless, and Venus was left with 100 million USD in bad debt.

### 7.4 Mitigations

- Liquidation caps -- limit total value that can be liquidated in a single transaction or time window
- Liquidation delays -- introduce a short delay between a position becoming liquidatable and liquidation executing
- Diverse oracle sources -- do not rely on a single exchange's price for liquidation decisions
- Graceful liquidation -- liquidate partial positions rather than entire positions at once

---

## 8. Summary

| Attack Class | Mechanism | Example Incident | Estimated Loss |
|---|---|---|---|
| Flash Loan Manipulation | Borrow capital, manipulate price, exploit protocol, repay | bZx (2020) | 954,000 USD |
| Flash Loan at Scale | Same mechanism, larger capital, incentivised minting | Pancake Bunny (2021) | 45,000,000 USD |
| Oracle Manipulation (sustained) | Move spot price over time, exploit collateral valuation | Mango Markets (2022) | 114,000,000 USD |
| Lending Oracle Manipulation | Inflate collateral value via oracle, borrow against it | Cream Finance (2021) | 130,000,000 USD |
| Sandwich Attacks | Front-run and back-run large AMM trades | Systematic MEV extraction | 1,380,000,000 USD (2020-2023) |
| Wash Trading | Circular trading to farm protocol rewards | LooksRare (2021-2022) | Billions in artificial volume |
| Governance Manipulation | Flash loan voting power, pass malicious proposal | Beanstalk (2022) | 182,000,000 USD |
| Liquidation Manipulation | Crash collateral price, trigger and front-run liquidations | Venus Protocol (2021) | 100,000,000 USD |

---

## 9. Conclusion

DeFi market manipulation is not a single attack type -- it is a category of behaviours that exploit the specific properties of decentralised financial infrastructure: transparent pending transactions, AMM pricing mechanics, oracle dependencies, instant capital availability through flash loans, and governance systems with insufficient safeguards.

What distinguishes DeFi manipulation from traditional market manipulation is not the intent -- extracting value through artificial price movements -- but the infrastructure. Flash loans remove the capital barrier. Pseudonymity reduces accountability. Atomic transactions allow complex multi-step attacks to execute in seconds. Permissionless protocols cannot distinguish between legitimate and manipulative trading.

The security practitioner working in DeFi needs to understand market manipulation not just as a financial concept but as a technical attack surface. Every protocol design decision -- how oracles are sourced, how collateral is valued, how governance votes are weighted, how liquidity pools are structured -- has a manipulation attack surface attached to it. Understanding that surface is the prerequisite for securing it.

---

## References

- Flashbots -- MEV Explore and sandwich attack data
- EigenPhi -- DeFi attack taxonomy and incident data
- bZx post-mortem -- February 2020
- Pancake Bunny post-mortem -- May 2021
- Mango Markets incident analysis -- October 2022
- Cream Finance post-mortem -- October 2021
- Beanstalk Farms governance attack post-mortem -- April 2022
- Venus Protocol incident report -- May 2021
- Compound DAI oracle incident -- November 2020
- Chainalysis -- NFT wash trading report 2022
- CoW Protocol -- batch auction documentation
- Uniswap -- TWAP oracle documentation
