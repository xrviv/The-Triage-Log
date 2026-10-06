---
title: "Trustodial: Where the Bitcoin Community Lands on Spark Custody"
description: "Custodial, self-custodial, or something in between? A survey of the public debate on Spark, and what the 2026 exit tests changed"
date: 2026-10-06 12:20:00 +0800
categories: [Blogging, Tech]
tags: [walletscrutiny, bitcoin, lightning, spark, custody]
author: bob
---

# Trustodial: where the Bitcoin community lands on Spark custody

## The question

Since mid-2025 a growing number of Lightning wallets, among them Wallet of Satoshi, Blink, Trustless and Cake Wallet, have moved their Lightning balances onto Spark, the Lightspark-designed layer that the Breez SDK exposes as "Breez SDK Spark". Each of these wallets tells users some version of "your keys, your coins". Reviewers who have to put a label on such a wallet face a question that sounds simple and is not: is a Spark balance custodial, self-custodial, or something else?

The question matters for WalletScrutiny because our verdict system is binary at the custody boundary. A product is either custodial, in which case it fails and nothing else about it is examined, or it is not, in which case we go on to ask whether the source is available and the build reproducible. Spark was built to live in the space between those two words, and the public debate of the last fifteen months is mostly an argument about which side of the line that space belongs to.

This essay summarises that debate: what the three camps say, who is in each, what the 2026 field tests changed, and what it means for the people holding the balance.

## What Spark is, in one paragraph

Spark is a statechain-style system. A user's funds sit in on-chain UTXOs that are controlled jointly by the user and a small set of Spark Operators, currently two, Lightspark and Flashnet. Ownership of a "leaf" (a slice of such a UTXO) is transferred off-chain by having the operators re-sign with the new owner and delete their share of the old key. The security claim is that the operators cannot spend a user's funds, and that if the operators disappear the user can broadcast a pre-signed chain of transactions to pull the leaf back on chain after a timelock. That last step is the "unilateral exit", and almost everything in the custody debate turns on how real it is in practice.

## Camp one: "fully self-custodial"

The vendors use the strongest word available. Spark's own announcement of the Wallet of Satoshi launch on 2025-07-01 called the product "fully self-custodial" (<https://x.com/spark/status/1940168641301119094>). Flashnet, one of the two operators, had been describing Spark since 2024-10-25 as "the most trust-minimized L2 for Bitcoin (after Lightning)" holding "non-custodial BTC" (<https://x.com/flashnet/status/1849864396413280566>). Wallet of Satoshi, which had been a plainly custodial wallet for years, announced on 2025-10-08 that it was "now self-custody" (<https://x.com/walletofsatoshi/status/1975773876186718236>).

The most careful statement of the vendor position comes from Lightspark's CTO Kevin Hurley, replying to critics on 2025-07-02. Spark, he wrote, is a "1-of-n" system in which operators need to be "honest only at the point of transfer", and users are "free to unilaterally exit at any point and the operators have no ability to censor or stop them" (<https://x.com/kphur/status/1940228052107280726>). Lightspark's own blog post of 2025-07-28, "Unilateral Exit is Now Live", repeats the claim while conceding the cost: exits are deliberately slower and more expensive than cooperative withdrawals, because, in the post's phrase, "safety > UX".

Two things are notable about this camp. First, its members are all commercially invested: the protocol designer, the operators and the wallets that have migrated to it. Second, even inside the camp the language softens on contact with detail. Wallet of Satoshi's disclosure document admits that unilateral exits "may involve longer processing times and higher costs". Blink, which runs Spark accounts alongside its old custodial ones, published the only public mainnet exit test in July 2026 and announced it with "no operator cooperation needed" (<https://x.com/blinkbtc/status/2077402656650256783>), but the accompanying blog post is far more hedged, as the next sections show.

## Camp two: neither word fits

The largest group, and the one that contains most of the independent developers and reviewers, refuses both labels.

Matt Corallo set the tone on the first day of the debate. On Stacker News on 2025-07-01 he wrote that Spark is "much better than a classic custodial wallet" but "still requires fully trusting the operator", and added the observation that would become the centre of the 2026 discussion: "if you put too little in it it's very much not non-custodial cause you can't unilaterally exit ... without a fairly high fee" (<https://stacker.news/items/1020261>). The next day on X he was explicit about the vocabulary: "it doesn't make sense to call Spark 'custodial' (something I carefully avoided doing!) ... But that level of trust is also drastically different from what constitutes 'self-custodial'" (<https://x.com/TheBlueMatt/status/1940236872317640820>).

Janusz offered the cleanest ladder the same day: "lightning = trust minimized; spark = trusted, but you have immediate custody if operators are honest; liquid = custodial" (<https://x.com/januszg_/status/1940210878999036119>). Arkade, which builds the competing Ark protocol and therefore has its own interest, argued on 2025-07-03 that "Spark transactions never really finalize from the view of users", because the user cannot verify that the operators deleted the old key (<https://x.com/arkade_os/status/1940767805848408395>).

Shinobi gave the position its name in Bitcoin Magazine on 2025-07-07, borrowing Justin Shocknet's coinage "trustodial" for the title "Trustodial: An Ontological Dilemma". His conclusion was that Spark is non-custodial if the operators are honest, but that the user has "no way to trustlessly verify" that they were, and that "there is no real clear cut answer". Seth for Privacy, writing for the same magazine on 2025-10-23 about Spark and Ark together, put it as finality that "exists but is not provable", and warned that the exit path was "likely pricing out smaller amounts". Bitcoin Layers, the most systematic of the reviewers, rates Spark's custody trust assumption "Low" but its operator liveness risk "Very High", and records the two-operator set.

The word "trustodial" has since been adopted by k00b and Scoresby on Stacker News and by Seth, Shinobi and Janusz in print, which makes it the closest thing the debate has to a consensus label.

Two 2026 contributions moved this camp from theory to measurement. Federico Rivi's Bitcoin Train piece of 2026-08-22, "Spark: una self-custody condizionata" (conditional self-custody), argued that most consumer Spark wallets cannot perform an exit without the operators being online to serve the exit data, and that until they can, "self-custodial" is marketing. The anonymous site spark.exposed, last updated 2026-09-11, audited twelve consumer wallets built on Spark and found "no complete in-app operatorless exit" in any of them; it also points at the operators' identity checks as a "kill switch", and concludes that the wallets are effectively custodial in practice even if the protocol is not.

## Camp three: "a fake L2"

The hard critics are fewer but loud. Justin Shocknet, who invented "trustodial" on Stacker News on 2025-07-02, did not intend it as a compliment: Spark is "a fake L2", "not better than regular custodial", and "it's already a trust violation for them to lie about it". By 2025-10-08 he had sharpened it to "it's not a protocol, it's a centralized exchange". Megalithic, replying to Stephan Livera's SLP700 on 2025-10-31, called the self-custodial marketing "an obvious scam", noting that an exit to Lightning requires Lightspark to custody the funds on its own nodes for the hop, that the SSP (service provider) code is closed, and that the operator set is in effect one company's servers (<https://x.com/MegalithicBTC/status/1984335021973598257>; the same points are filed as buildonspark/spark issue #64 of 2025-09-15). Rizful and DarthCoin on Stacker News in November 2025 made the same point from the user side: every operation goes through a Lightspark-controlled API, and a "corporate network" is not a protocol.

A side issue feeds this camp without being about custody. On 2025-10-08 instagibbs showed that every Spark balance is public on the sparkscan explorer and that a user's Spark public key leaks from any bolt11 invoice they issue (<https://x.com/theinstagibbs/status/1975944960160465405>). That is a privacy failure rather than a custody one, but it reinforces the critics' picture of a system run as a hosted service.

## What the 2026 tests changed

Until mid-2026 the argument was about whether the exit could in principle be done. Two field tests settled that question and raised a different one.

Blink's case study, published 2026-07-13 (<https://www.blink.sv/blog/case-study-spark-unilateral-exit-on-bitcoin-mainnet>), forced 100,000 sats out of Spark on mainnet using only the seed and a recovery bundle saved in advance. It worked. But the numbers are the point. The balance was spread over 22 leaves. At 1 sat/vB only 4 of them were worth exiting; the other 18, holding 9,888 sats, were abandoned as dust because the exit transactions would have cost more than the leaves contained. The user recovered 89,668 sats. Had they insisted on exiting every leaf, fees would have eaten roughly 79 % of the balance. The recovery bundle has to be fetched and saved while the operators are still online, because the exit data lives with them. Blink's own summary is "not trustless" and "a fire escape, not a door".

Francis Pouliot's test of September 2026, which Leo discussed with him on X on 2026-09-21/22, came out harsher at almost the same fee rate (1.25 sat/vB against Blink's 1 sat/vB); the difference lies in how each balance was split into leaves, not in the fee environment. At 1.25 sat/vB he lost about 30 % of 100,000 sats, with about 18,000 sats left to the operators as uneconomical leaves; at 10 sat/vB exiting all leaves would have cost 1.4 million sats, fourteen times the balance. Leo's response was a rule rather than a label: the honest states a wallet can be in are "0 % / 100 % / else", and a Spark wallet should show "an exit and a sats price tag at all times without internet".

These tests do not contradict the vendors. Hurley said the operators cannot stop an exit, and they could not. They confirm Corallo's 2025 caveat instead: below some balance, which depends on the fee rate and the leaf structure, the exit is a right the user cannot afford to exercise. A right you cannot afford is, for the person holding it, no right at all.

## Where that leaves the question

Three conclusions follow from the record.

First, the protocol-level question is settled in the middle and is unlikely to move. Nobody independent calls Spark self-custodial without qualification, and the people who call it custodial flat out are making a point about honesty in marketing rather than about key control. "Trustodial", "conditional self-custody", "trusted but not custodial": the labels differ, the content is the same. The operators cannot spend your funds; they can refuse to serve you; you can leave, but slowly, expensively and only if you prepared while they were still cooperating.

Second, the debate has shifted from the protocol to the wallet. Every 2026 source, Rivi, spark.exposed, Blink and Francis, measures the same three things: does this app perform the unilateral exit at all, does it store the exit data locally so the exit works when the operators are gone, and does it tell the user what the exit will cost. By those tests the twelve consumer wallets audited by spark.exposed all fail the first or second, and Blink comes closest: a separate open-source tool performs the exit and prices each leaf, but the app itself does neither yet, and saving the exit data inside the app is planned rather than shipped. The custody status of a Spark wallet is therefore a property of the wallet, not of Spark.

Third, the balance size is part of the answer. Corallo said it in July 2025 and Blink measured it in July 2026: for a balance of a few hundred thousand sats or less, spread across the number of leaves a normal payment history produces, the exit returns a fraction of the money. For the typical user of a consumer Lightning wallet that is the whole balance. A label that is accurate for a whale and false for everyone else is false for the product.

## Conclusion: what this means for Bob and Lucy

Labels are easier to argue about than outcomes, so take two users and the day the operators stop answering, whether through an outage, a shutdown or a refusal to serve one particular customer.

**Bob** keeps 50,000 sats in a typical consumer Spark wallet. Like the twelve wallets spark.exposed examined, his app offers only the cooperative withdrawal, and it has never saved any exit data on his phone. When the operators go quiet, his withdraw button simply fails. He restores his twelve words in another wallet and his on-chain coins appear, but the Spark balance does not, because a Spark wallet's current leaves cannot be discovered from the seed alone. Nobody has stolen anything: the operators cannot spend Bob's sats. But he cannot reach them either, and he has no move of his own to make. For Bob, today, the balance is worth exactly what the operators decide to release, which is Leo's "0 %".

**Lucy** keeps 100,000 sats on Spark too, but she has done what Blink's engineer did: she runs the open-source exit tool and refreshes her recovery bundle while the operators are still online. When they disappear, she can leave without asking anyone. It is not quick or free. At 1 sat/vB she pays about 8,400 sats in fees from a separate on-chain coin, waits through the chain of confirmations and a timelock of roughly ten days, and abandons the dust leaves that cost more to exit than they hold. About 89,700 sats arrive in her own wallet. If fees are ten times higher on the day she needs to leave, more leaves fall below the line and the fees on the rest grow tenfold; Francis lost about 30 % at almost the same low fee rate, because his balance was split differently. Lucy is in Leo's "else": her coins are hers, minus a price that depends on the fee market and on how her balance happens to be cut into leaves.

The difference between Bob and Lucy is not the protocol. Both used Spark. It is three things the wallet does or does not do: offer the unilateral exit, keep the exit data on the device, and show what leaving would cost. Bob's wallet does none of them; Lucy did all three by hand, and no wallet app yet does them for her.

That is where the debate really stands. The honest critics and the honest vendors agree on the facts and disagree on the word. "Trustodial" is a fair name for the protocol, but a user does not hold a protocol; they hold a balance in a particular app. Before trusting a Spark wallet with more than spending money, ask it the three questions. If it cannot answer them, you are Bob.

## Sources and access note

X search and timelines were not available while researching this piece (login wall; the syndication endpoint rate-limits; nitter instances are dead). Individual tweets cited above were resolved through a read-only mirror API; discovery was via web search, the Stacker News GraphQL API, Bitcoin Magazine, the btc++ newsletter, Bitcoin Train, spark.exposed and blink.sv. The 2026-09-21/22 Francis Pouliot / Leo Wandersleb thread could not be located without X search and is cited from a summary of it.
