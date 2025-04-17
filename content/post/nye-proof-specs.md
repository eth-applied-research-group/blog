---
date: "2025-04-15T19:30:32+02:00"
title: "New Year's Eve-Proofing Your Specifications"
---

On an unsuspecting [New Year's Eve](https://github.com/openssl/openssl/commit/4817504d069b4c5082161b02a22116ad75f822b1), the few lines of C code shown below nearly crippled the Internet, raking up half a billion dollars in damages.

{{< figure src="/img/nye-proof-specs/heartbeat-code.png" >}}

## The heartbleed bug

The code in question implements a new ["Heartbeat" specification](https://www.rfc-editor.org/rfc/rfc6520) for the [TLS encryption protocol](https://en.wikipedia.org/wiki/Transport_Layer_Security), aiming to solve an important bottleneck: establishing a new TLS connection is expensive. The specification proposes that the sender post an arbitrary "heartbeat" message to the recipient. The recipient interprets this as a signal to keep the connection alive and confirms by echoing the message back to the sender.

{{< figure src="/img/nye-proof-specs/say-bird.jpg" caption="Credit: xkcd" >}}

Neat. But the code had a fatal flaw: it trusts that the sender reports correct length of the message.

The recipient didn't check.

This gives the sender an opportunity to blatantly lie about the message length, tricking the recipient into leaking extra information from memory back to the sender.

{{< figure src="/img/nye-proof-specs/say-hat.jpg" >}}

The now infamous bug, cleverly named [_Heartbleed_](https://www.heartbleed.com/), lets a malicious sender peek behind the TLS curtain and expose contents from memory. In a twist of cosmic irony, it often reveals exactly the information that TLS is designed to protect—such as private keys or login credentials.

Here is an exploited memory dump with sensitive information exposed:

{{< figure src="/img/nye-proof-specs/memory-dump.png" caption="Credit: @markloman" >}}

Interestingly, the specification casually notes to "silently discard" this edge case:

{{< figure src="/img/nye-proof-specs/highlighted-specs.jpg" >}}

How could we better nudge implementation do what it says on the tin?

More unit tests? Yes.

No deployments on New Year's Eve? A necessary sacrifice, sure.

But the _why_ is rather hidden in plain sight. Throughout a specification, we bury its structural integrity —its _invariants_—in normative prose.

What if we didn't?

## First-class spec invariants

Fortunately, Ethereum's specs are [executable](https://github.com/ethereum/execution-specs), which strengthens their reliability. But there's still a clear edge to having explicitly defined invariants.

First, we get **standardized errors** for free. Currently, errors are a fragmented mess across clients.

For instance, consider the following excerpt from [EIP-7623](https://eips.ethereum.org/EIPS/eip-7623):

> Any transaction with a gas limit below `21000 + TOTAL_COST_FLOOR_PER_TOKEN * tokens_in_calldata` or below its intrinsic
> gas cost (take the maximum of these two calculations) is considered invalid. This limitation exists because transactions
> must cover the floor price of their calldata without relying on the execution of the transaction.
> There are valid cases where `gasUsed` will be below this floor price, but the floor price needs to be reserved in
> the transaction gas limit.

For the exact same spec, geth has [two errors](https://github.com/ethereum/go-ethereum/blob/13b157a461c88678cd4e15ca005e7b45d823431b/core/error.go#L77-L83), while the [testing framework](https://github.com/ethereum/execution-spec-tests) has [just one.](https://github.com/ethereum/execution-spec-tests/issues/1412)

Plus, in the absence of standard error codes, the testing framework falls back on [client-specific error messages](https://github.com/ethereum/execution-spec-tests/blob/main/src/ethereum_clis/clis/geth.py) to detect errors:

{{< figure src="/img/nye-proof-specs/exception-mapper.png" >}}

A typo or slight change in the client error message can lead to outcomes ranging from a [benign false positive](https://github.com/ethereum/execution-spec-tests/issues/1412) to a gaping security hole.
The mechanism is so abysmally delicate that you can almost hear it creaking.

First-class invariants in specs eliminate the guesswork:

| Field                 | Example                                                                                                           |
| --------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Identifier**        | `7623-01` [EIP namespaced, enumerated]                                                                            |
| **Description**       | Guarantees transactions reserve a minimum gas amount needed to cover revised calldata costs.                      |
| **Formal Expression** | `∀ Tx: gas_limit(Tx) ≥ max( intrinsic_gas_cost(Tx), 21000 + TOTAL_COST_FLOOR_PER_TOKEN × tokens_in_calldata(Tx))` |
| **Scope**             | All Ethereum transactions.                                                                                        |
| **Rationale**         | Prevents processing transactions with insufficient gas to protect against potential execution or security issues. |

The outcome is a precise protocol-wide violation. Its testable - across clients, across versions, across decades.

{{< figure src="/img/nye-proof-specs/better-error.png" >}}

Second, explicitly defined invariants in both the specs and their executable versions make **formal verification simpler**, helping spot conflicts as the protocol grows complex.
Hopefully, implementation teams will keep enriching specs with any new invariants they find along the way—it's a win-win.

Specs shouldn't whisper invariants. They should _yell_ them.

## References

- [Diagnosis of the OpenSSL Heartbleed Bug](https://web.archive.org/web/20141015215508/http://blog.existentialize.com/diagnosis-of-the-openssl-heartbleed-bug.html)
- [Heartbleed, Bruce Schneier](https://www.schneier.com/blog/archives/2014/04/heartbleed.html)
- [Critical crypto bug exposes Yahoo Mail, other passwords Russian roulette-style](https://arstechnica.com/information-technology/2014/04/critical-crypto-bug-exposes-yahoo-mail-passwords-russian-roulette-style/)
- [Relevant xkcd](https://xkcd.com/1354/)
