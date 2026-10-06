---
title: "Trustodial: Where the Bitcoin Community Lands on Spark Custody"
description: "Custodial, self-custodial, or something in between? A survey of the public debate on Spark, and what the 2026 exit tests changed"
date: 2026-10-06 12:20:00 +0800
categories: [Blogging, Tech]
tags: [walletscrutiny, bitcoin, lightning, spark, custody]
author: bob
image:
  path: /assets/img/posts/2026-10-06-spark-custody/spark-leaf-tree.svg
  alt: "How a Spark balance is held: one on-chain UTXO split into leaves, each locked to your key plus the operators' key, with a normal co-signed path and a slow pre-signed exit path."
---

# Trustodial: where the Bitcoin community lands on Spark custody

## The question

Since mid-2025 Lightning wallets such as Wallet of Satoshi, Blink, Trustless and Cake Wallet have moved their Lightning balances onto Spark, the Lightspark-designed layer that the Breez SDK exposes as "Breez SDK Spark". Each tells users some version of "your keys, your coins". So is a Spark balance custodial, self-custodial, or something else?

The question matters for WalletScrutiny because our verdict system is binary at the custody boundary: a custodial product fails, a non-custodial one goes on to the source and build checks. Spark was built to live between those two words, and the public debate of the last fifteen months is an argument about which side that space belongs to.

## What Spark is, in one paragraph

Spark is a statechain-style system. Funds sit in on-chain UTXOs controlled jointly by the user and a small set of operators, currently two: Lightspark and Flashnet. A "leaf" (a slice of such a UTXO) changes owner off-chain when the operators re-sign with the new owner and delete their share of the old key. The operators cannot spend a user's funds, and if they disappear the user can broadcast a pre-signed chain of transactions to pull the leaf back on chain after a timelock. That "unilateral exit" is what the whole custody debate turns on.

## What are Operators?

Spark Operators are the companies that run Spark's signing servers; today there are two, Lightspark and Flashnet. Together they hold the other half of the key that locks every leaf, so no payment, transfer or cooperative withdrawal happens without their signature alongside yours. When a leaf changes owner, the operators sign for the new owner and are supposed to delete their share of the old key; Lightspark calls this a "1-of-n" model, because a single operator that honestly deletes its share is enough to keep the transfer final. They cannot move your sats on their own: Blink's case study notes that theft would need every operator to collude. What they can do is stop answering, and because they also serve the data a wallet needs to build an exit, an app that never saved that data leaves its user with nothing to broadcast.

## Three camps

**"Fully self-custodial": the vendors.** Spark announced the Wallet of Satoshi launch on 2025-07-01 as "fully self-custodial" (<https://x.com/spark/status/1940168641301119094>); Flashnet calls Spark "the most trust-minimized L2 for Bitcoin (after Lightning)" (<https://x.com/flashnet/status/1849864396413280566>); Wallet of Satoshi announced it was "now self-custody" (<https://x.com/walletofsatoshi/status/1975773876186718236>). Lightspark's CTO Kevin Hurley gave the careful version: operators need to be "honest only at the point of transfer", and users are "free to unilaterally exit at any point" (<https://x.com/kphur/status/1940228052107280726>). Everyone in this camp is commercially invested, and their language softens with detail: Wallet of Satoshi's disclosure admits exits "may involve longer processing times and higher costs".

**"Neither word fits": most independent developers and reviewers.** Matt Corallo called Spark "much better than a classic custodial wallet" but said it "still requires fully trusting the operator", adding: "if you put too little in it it's very much not non-custodial cause you can't unilaterally exit ... without a fairly high fee" (<https://stacker.news/items/1020261>). On X: "it doesn't make sense to call Spark 'custodial' ... But that level of trust is also drastically different from what constitutes 'self-custodial'" (<https://x.com/TheBlueMatt/status/1940236872317640820>). Janusz: "spark = trusted, but you have immediate custody if operators are honest" (<https://x.com/januszg_/status/1940210878999036119>). Arkade, a competitor, argued that "Spark transactions never really finalize from the view of users" (<https://x.com/arkade_os/status/1940767805848408395>). Shinobi named the position in Bitcoin Magazine, "Trustodial: An Ontological Dilemma": non-custodial if the operators are honest, with "no way to trustlessly verify" it. Seth for Privacy warned the exit was "likely pricing out smaller amounts"; Bitcoin Layers rates custody trust "Low" but operator liveness risk "Very High". In 2026, Bitcoin Train called it "conditional self-custody", and spark.exposed found "no complete in-app operatorless exit" in any of the twelve consumer wallets it audited. "Trustodial" is the closest thing the debate has to a consensus label.

**"A fake L2": the hard critics.** Justin Shocknet, who coined "trustodial", meant it as an insult: "not better than regular custodial", and later "it's not a protocol, it's a centralized exchange". Megalithic called the self-custodial marketing "an obvious scam", pointing at closed service-provider code and an operator set that is in effect one company's servers (<https://x.com/MegalithicBTC/status/1984335021973598257>). Separately, instagibbs showed that Spark balances are public on the sparkscan explorer and that a user's Spark key leaks from their invoices (<https://x.com/theinstagibbs/status/1975944960160465405>): a privacy failure, not a custody one, but it feeds the same picture.

## What the 2026 tests changed

Two field tests settled whether the exit works and raised a harder question: what it returns.

Blink's case study (<https://www.blink.sv/blog/case-study-spark-unilateral-exit-on-bitcoin-mainnet>, announced at <https://x.com/blinkbtc/status/2077402656650256783>) forced 100,000 sats out on mainnet with the seed and a recovery bundle saved in advance. The balance sat in 22 leaves. At 1 sat/vB only 4 were worth exiting; the other 18, holding 9,888 sats, were abandoned as dust. 89,668 sats arrived. Exiting every leaf would have cost about 79 % in fees, and the bundle has to be saved while the operators are still online. Blink's own words: "not trustless", "a fire escape, not a door".

![Where 100,000 sats go in Blink's mainnet exit at 1 sat/vB: 89,668 arrive, 9,888 are left behind as dust leaves, 444 go to sweep fees, and 8,388 sats of exit fees come from a separate coin.](/assets/img/posts/2026-10-06-spark-custody/where-lucys-sats-go.svg)

Francis Pouliot's test of September 2026, which Leo discussed with him on X on 2026-09-21/22, came out harsher at almost the same fee rate (1.25 sat/vB against Blink's 1 sat/vB): about 30 % of 100,000 sats lost, about 18,000 left behind as uneconomical leaves. The difference lies in how each balance was split into leaves. At 10 sat/vB, exiting all his leaves would have cost 1.4 million sats. Leo's response was a rule rather than a label: "0 % / 100 % / else", and a wallet should show "an exit and a sats price tag at all times without internet".

Neither test contradicts the vendors: the operators could not stop the exits. They confirm Corallo's caveat instead. Below some balance, set by the fee rate and the leaf structure, the exit is a right the user cannot afford to use.

![The five steps of a Spark exit: save the exit data while operators are online, operators go quiet, broadcast the pre-signed transactions one level per block, wait out a 10 to 14 day timelock, then sweep to your own address.](/assets/img/posts/2026-10-06-spark-custody/exit-timeline.svg)

## Where that leaves the question

- **The protocol question is settled in the middle.** Nobody independent calls Spark self-custodial without qualification. The operators cannot spend your funds; they can refuse to serve you; you can leave, but slowly, expensively, and only if you prepared while they still cooperated.
- **The debate has moved from the protocol to the wallet.** Every 2026 source asks the same three things of an app: does it perform the unilateral exit, does it keep the exit data on the device, and does it show what the exit will cost? spark.exposed found all twelve audited wallets fail the first or second. Blink comes closest, with a separate open-source tool that performs the exit and prices each leaf, but the app itself does neither yet, and saving the exit data in the app is planned rather than shipped.
- **Balance size is part of the answer.** For a few hundred thousand sats or less, spread over the leaves a normal payment history produces, the exit returns a fraction of the money. A label that is accurate for a whale and false for everyone else is false for the product.

## Conclusion: what this means for Bob and Lucy

Take two users on the day the operators stop answering.

**Bob** keeps 50,000 sats in a typical consumer Spark wallet that offers only the cooperative withdrawal and has never saved exit data on his phone. His withdraw button fails. He restores his twelve words elsewhere and his on-chain coins appear, but the Spark balance does not, because current leaves cannot be discovered from the seed alone. Nothing is stolen; he simply cannot reach it. That is Leo's "0 %".

**Lucy** keeps 100,000 sats on Spark but runs the open-source exit tool and keeps her recovery bundle fresh. She can leave without asking anyone, at a price: at 1 sat/vB, about 8,400 sats in fees from a separate on-chain coin, roughly ten days of confirmations and timelock, and the dust leaves abandoned. About 89,700 sats arrive. At ten times the fee rate, more leaves fall below the line. That is Leo's "else".

![Three wallets each show 100,000 sats. If the operators stop answering, Lucy can recover about 89,700, Francis's test recovered about 70,000, and Bob can recover nothing on his own.](/assets/img/posts/2026-10-06-spark-custody/same-balance-different-recovery.svg)

Both used Spark. The difference is three things the wallet does or does not do: offer the exit, keep the exit data, show its cost. Bob's wallet does none of them; Lucy did all three by hand, and no wallet app yet does them for her. "Trustodial" is a fair name for the protocol, but a user holds a balance in a particular app. Before trusting a Spark wallet with more than spending money, ask it the three questions. If it cannot answer them, you are Bob.

## Sources and access note

X search was not available while researching this piece; individual tweets were resolved through a read-only mirror, and discovery was via web search, Stacker News, Bitcoin Magazine, the btc++ newsletter, Bitcoin Train, spark.exposed and blink.sv. The 2026-09-21/22 Francis Pouliot / Leo Wandersleb thread is cited from a summary of it.
