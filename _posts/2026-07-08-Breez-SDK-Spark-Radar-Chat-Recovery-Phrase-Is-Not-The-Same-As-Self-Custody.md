---
title: "Breez SDK Spark and Radar Chat: Recovery Phrase Is Not the Same as Self-Custody"
description: "Tracing a Bitcoin wallet's exit path through its actual source code, not its marketing copy"
date: 2026-07-08 21:09:00 +0800
categories: [Blogging, Tech]
tags: [reproducible-builds, walletscrutiny, forensics, bitcoin, lightning]
image:
  path: /assets/img/posts/2026-07-08-radar-custody/banner.png
  alt: "Diagram showing Radar Chat's custody model: recovery phrase restores wallet access but does not provide an independent Bitcoin exit, because neither cooperative withdrawal nor unilateral exit is available to the user."
---

# Breez SDK Spark and Radar Chat: Recovery Phrase Is Not the Same as Self-Custody

A wallet giving you a 12-word backup phrase and a wallet giving you actual control over
your Bitcoin sound like the same thing. They aren't, and the gap between them is
exactly what this post is about.

The trigger was a public dispute, not a hunch. A Bitcoin developer,
[@notgrubles](https://x.com/notgrubles/status/2074624676978962867), posted that no wallet
built on Spark — a newer Bitcoin layer-2 system — actually implements "unilateral exit"
in practice, meaning users fully trust a third party regardless of what the underlying
protocol claims to support. [Francis Pouliot](https://x.com/francispouliot_/status/2074651604943245529),
CEO of Bull Bitcoin, agreed: "a Spark wallet is pretty much a custodial wallet unless
you can actually exit unilaterally if the servers go down."
[WalletScrutiny](https://walletscrutiny.com/) picked one Spark-based wallet, **Radar**
("Radar: Chat & Bitcoin," a Signal fork adding Bitcoin payments), and went and checked.
Key technical claims below are backed by exact source-code citations, not a press
release — a few broader statements (repo-wide searches that turned up nothing, issue
tracker checks) describe the search performed rather than a single line, since there's
no specific line a "no result" points to.

---

## Part 1 — Two very different ways to "get your money out"

Spark works by having your own keys plus a separate Spark service/operator
infrastructure jointly authorize your everyday transfers. It's not even a simple "one
company cooperates with you" setup: Breez SDK Spark's default configuration pools
*three* independent operators — Lightspark, Breez's own, and Flashnet —
[confirmed directly in the SDK's default config](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/config.rs#L99),
and requires 2 of those 3 to agree
([`split_secret_threshold: 2`](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/config.rs#L64)),
on top of your own key. Whatever the exact internal shape, the practical fact holds:
your key alone isn't enough for a normal transaction; outside infrastructure has to
cooperate too. That single fact splits "getting your bitcoin onto a normal on-chain
address" into two completely different operations, and mixing them up is the easiest
way to overstate what a wallet actually does:

- **Cooperative withdrawal**: you ask to withdraw, and the service provider helps build
  and broadcast the transaction. This is the "normal" path — but it only works as long
  as that service is running and willing to help you.
- **Unilateral exit**: you move your funds to a normal Bitcoin address entirely on your
  own, using pre-signed exit transactions, with *no* cooperation needed from the service
  at all — even if it disappeared or refused to help. This is the one that would
  actually make a wallet self-custodial in the strict sense: exclusive user control, no
  third party's permission required.

A wallet can support neither of these, one, or both. Marketing copy tends to blur that
distinction; source code doesn't.

## Part 2 — Where to look, and what to look for

Radar ships on two platforms with two separate, public repositories:
[`radar-labs/radar-android`](https://github.com/radar-labs/radar-android) (Kotlin) and
[`radar-labs/Radar`](https://github.com/radar-labs/Radar) (Swift, the iOS app that's
actually live on the App Store right now). Both depend on the same underlying payment
engine, [Breez SDK Spark](https://github.com/breez/spark-sdk), pinned at the exact same
version (`0.14.0`) on both platforms.

The investigation had two layers, and both had to be checked, because a wallet can fail
either one independently:

1. **Does Radar's own app code ever call a withdrawal or exit function?** This means
   cloning both repos and grepping every source file — not just the payments code — for
   the vocabulary of moving funds out: `withdraw`, `exit`, `unilateral`, `leaf` (Spark's
   term for an individual UTXO-like unit inside the wallet), `refund`, `recover`. Also
   worth checking: every string shown in the app's UI, and the GitHub issue tracker, in
   case an exit feature is planned but unshipped.
2. **Does the SDK even expose that capability to the app in the first place?** Android
   and iOS apps talk to Breez SDK Spark through auto-generated bindings (via a tool
   called [UniFFI](https://mozilla.github.io/uniffi-rs/)), which take a Rust function and
   mechanically produce a matching Kotlin or Swift method with the same name. If a Rust
   function in the SDK isn't explicitly marked for export, there is *no* way for the app
   to call it — full stop, regardless of what the app's developers want. So the second
   check is: clone `breez/spark-sdk` at the exact tag both apps pin (`0.14.0`), and read
   which functions are actually marked exported.

## Part 3 — What the code actually shows

**Cooperative withdrawal exists in the SDK — Radar's own code just never reaches it.**
The SDK does support sending Bitcoin to a normal on-chain address: its `prepare_send_payment`
function has a dedicated
[`InputType::BitcoinAddress`](https://github.com/breez/spark-sdk/blob/0.14.0/crates/breez-sdk/core/src/sdk/payments.rs#L326)
branch, and `send_payment` follows through by calling
[`spark_wallet.withdraw(...)`](https://github.com/breez/spark-sdk/blob/0.14.0/crates/breez-sdk/core/src/sdk/payments.rs#L1390)
([defined here](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/wallet.rs#L1181)),
which calls [`coop_exit_service.coop_exit(...)`](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/wallet.rs#L1270) —
a real, working cooperative withdrawal, provided the Spark service cooperates. Radar's
own outgoing-payment code never touches this, on either platform:

- On iOS, the app's internal `PreparedTransaction` type
  ([`PaymentsImpl.swift#L1660`](https://github.com/radar-labs/Radar/blob/d997428ba36916d5466dc3451f31de703d2034be/SignalUI/Payments/PaymentsImpl.swift#L1660))
  has exactly two cases: `.lnurlPay` and `.bolt11`. There is no Bitcoin-address case at
  all. A stray one reaching that code path is treated as a bug
  (`owsFailDebug("Unexpected payment method for BOLT11 invoice.")`), not a valid outcome.
- On Android, `sendPayment()` in
  [`BreezSdkWrapper.kt`](https://github.com/radar-labs/radar-android/blob/4a6e9f8f2adf914c97f951c81c80ad41c710a26d/app/src/main/java/org/thoughtcrime/securesms/payments/BreezSdkWrapper.kt#L186)
  only calls the pair of functions meant for paying Lightning addresses, and its own
  internal helper throws an error the moment it's handed a Bitcoin address instead —
  before the request would ever reach the SDK's withdrawal function.

**Unilateral exit isn't reachable at all — and this time it's not Radar's choice.** The
underlying Spark protocol genuinely has a
[`unilateral_exit()`](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/wallet.rs#L1293)
function. But it lives inside an internal Rust crate that the SDK's own `breez-sdk-spark`
layer never re-exports through UniFFI at version `0.14.0`. Reading every function actually
marked for export (there are about 30, covering payments, contacts, deposits, lightning
addresses, and wallet sync) confirms `unilateral_exit` isn't among them. Even a
hypothetical, more ambitious version of Radar couldn't wire up a "recover my funds
independently" button today — the tool to build it with doesn't exist yet in the SDK
version both its apps depend on.

## Part 4 — The recovery phrase: real, but not what it looks like

Radar does have a genuine, working 12-word backup phrase system on Android — and this is
exactly the part worth being precise about, because it's the easiest thing to mistake for
"self-custodial."

On Android, wallet creation generates 16 bytes of device randomness
([`Entropy.generateNew()`](https://github.com/radar-labs/radar-android/blob/4a6e9f8f2adf914c97f951c81c80ad41c710a26d/app/src/main/java/org/thoughtcrime/securesms/payments/Entropy.java#L26))
and turns it into a standard 12-word [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
phrase, checking its own math at generation time — encode to words, decode back, assert
the bytes match
([`Entropy.asMnemonic()`](https://github.com/radar-labs/radar-android/blob/4a6e9f8f2adf914c97f951c81c80ad41c710a26d/app/src/main/java/org/thoughtcrime/securesms/payments/Entropy.java#L49)).
Typing the words back in reverses that exact operation. Clean, standard, textbook BIP-39.

iOS generates the phrase the same way, but its *restore* path is a genuinely interesting
detour. When you re-enter a phrase, iOS doesn't decode it back into the original random
bytes the way Android does — it runs the words through
[`MnemonicSwift.Mnemonic.deterministicSeedBytes(...)`](https://github.com/radar-labs/Radar/blob/d997428ba36916d5466dc3451f31de703d2034be/SignalUI/Payments/PaymentsImpl.swift#L412),
which traces back to a
[`PBKDF2SHA512` call](https://github.com/zcash/swift-bip39/blob/2d7f3e76e904621e111efada6cc9575f39543bb2/MnemonicSwift/Mnemonic.swift#L119-L135)
inside the underlying [`zcash/swift-bip39`](https://github.com/zcash/swift-bip39) library,
[implemented here](https://github.com/zcash/swift-bip39/blob/2d7f3e76e904621e111efada6cc9575f39543bb2/MnemonicSwift/PKC5.swift#L16-L27)
using Apple's CommonCrypto PBKDF2 with the SHA-512 PRF. That's a real,
standard BIP-39 operation — but it's the *seed-derivation* function (used for deriving
keys), not the *entropy-recovery* function Android uses. The output then gets stored
under the same field name used for the original entropy, and re-encoded into a brand-new
mnemonic to reconnect to the wallet — a different mnemonic than the one you actually
typed in. Whether that still reliably reconnects to the identical wallet isn't something
a source read alone can settle; it would need an actual device test. Filing that one
under "open question," not "bug" or "fine" — the honest answer is: unconfirmed.

Here's the distinction that matters regardless of that iOS detail: even in the best
case, where the phrase restores the wallet perfectly, that only gets you back into your
*same wallet*, using Radar itself or another app that speaks the Spark protocol the same
way. It proves the funds are yours in the sense of restoring access — it is not, by
itself, a way to move bitcoin to a normal on-chain address if the Spark service Radar
depends on ever goes offline or refuses to cooperate. Restoring a wallet with those
words still needs software that understands Spark, and that software still needs to talk
to a Spark service to figure out the current state of your funds and build a valid
withdrawal — the same cooperative arrangement from Part 1, which stops working the
moment the cooperating service is gone. That's what unilateral exit would solve, and as
shown above, it isn't there.

One more thing worth correcting up front, because it was the most tempting assumption to
get wrong: it's *not* Radar's own servers standing in as that cooperating party. Neither
app overrides the SDK's default configuration, which points at Lightspark's own
infrastructure (`api.lightspark.com`,
[confirmed in the SDK's default config](https://github.com/breez/spark-sdk/blob/0.14.0/crates/spark-wallet/src/config.rs#L52)).
The dependency on outside cooperation is real, it's just not Radar-the-company you'd be
depending on.

---

## Takeaways

- A working recovery phrase and independent fund recovery are two different claims. The
  first restores access to the same wallet through the same (or compatible) software;
  the second lets you get bitcoin out on a normal address with nobody's permission. A
  wallet can have the first without the second, and that's exactly what "self-custodial"
  marketing tends to gloss over.
- "The protocol supports X" and "the app lets you do X" are different sentences. Spark
  the protocol does support unilateral exit. Whether a specific wallet app can actually
  trigger it depends on whether the SDK's binding layer exports that function at all,
  and separately, on whether the app's own code ever calls it. Both checks are necessary;
  neither one alone is enough.
- Cross-check both platforms when a wallet ships on more than one. Radar's Android and
  iOS apps reach the same conclusion through genuinely different code paths — and the
  recovery-phrase restore mechanism turned out not to be identical between them at all,
  which wouldn't have surfaced from reading just one platform's source.
- Don't assume the "cooperating service" is the company whose name is on the app.
  Radar's apps don't configure their own Spark infrastructure — they use the SDK's
  default, which pools three completely different companies (Lightspark, Breez,
  Flashnet). Read the actual configuration instead of assuming.
- Some questions genuinely can't be settled by reading source alone. Whether iOS's
  restore path reliably reconnects to the identical wallet needs a device test this
  investigation didn't run. Saying "unconfirmed" honestly is more useful than guessing
  in either direction.

Full technical writeup, with every citation, lives on WalletScrutiny's
[Radar entry](https://walletscrutiny.com/mobile/com.cakelabs.signal/).
