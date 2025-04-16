---
date: "2025-04-15T19:30:32+02:00"
title: "New Year's Eve-Proofing Your Specifications"
---

On an unsuspecting [New Year's Eve](https://github.com/openssl/openssl/commit/4817504d069b4c5082161b02a22116ad75f822b1), the code shown below nearly crippled the Internet, resulting in damages estimated at half a billion dollars.

{{< figure src="/blog/img/nye-proof-specs/heartbeat-code.png" >}}

The code in question implements a new ["Heartbeat" specification](https://www.rfc-editor.org/rfc/rfc6520) for the [TLS encryption protocol](https://en.wikipedia.org/wiki/Transport_Layer_Security), aiming to solve an important bottleneck: establishing a new TLS connection is expensive. The specification proposes that the sender post an arbitrary "heartbeat" message to the recipient. The recipient interprets this as a signal to keep the connection alive and confirms by echoing the message back to the sender.

{{< figure src="/blog/img/nye-proof-specs/say-bird.jpg" caption="Credit: xkcd" >}}

Neat. Can you spot the bug?

The catch is that the recipient never checks whether the supplied length actually matches the message. This gives the sender an opportunity to blatantly lie about the message length, tricking the recipient into leaking extra information from memory back to the sender.

{{< figure src="/blog/img/nye-proof-specs/say-hat.jpg" >}}

The now infamous bug, cleverly named [_Heartbleed_](https://www.heartbleed.com/), lets a malicious sender peek behind the TLS curtain and expose contents from memory. In a twist of cosmic irony, it often reveals exactly the information that TLS is designed to protect—such as private keys or login credentials.

Here is an exploited memory dump with sensitive information exposed:

{{< figure src="/blog/img/nye-proof-specs/memory-dump.png" caption="Credit: @markloman" >}}

Interestingly, the specification casually notes to "silently discard" this edge case:

{{< figure src="/blog/img/nye-proof-specs/highlighted-specs.jpg" >}}

How could we better nudge implementation do what it says on the tin?

More unit tests? Yes.

No deployments on New Year's Eve? A necessary sacrifice, sure.

I’d argue that the _why_ is rather hidden in plain sight. Throughout a specification, the structural integrity of a system—its _invariant_—is often buried in normative prose.
This bug shows that it's all too easy to overlook invariants, especially on a _noisy_ New Year's Eve.

Instead, the format below lets a specification **YELL** its invariants:

| Field                 | Description                                                                                | Example                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| **Identifier**        | A unique reference that makes it easy to refer to the invariant.                           | `HB-01`                                                                                                           |
| **Description**       | A clear, human-readable explanation of what the invariant ensures.                         | The invariant guarantees that the declared length of a heartbeat message exactly matches its actual payload size. |
| **Formal Expression** | A formal version of the invariant provided in mathematical notation, pseudocode, or a DSL. | `∀ HeartbeatMessage: HeartbeatMessage.payload_length = payload_length`                                            |
| **Scope**             | Details where (or in which parts of the system) the invariant applies.                     | Heartbeat request and response.                                                                                   |
| **Rationale**         | Explains why the invariant is critical and what could go wrong if it’s violated.           | Prevents unintended side-effects caused by mismatch in the message and its declared length.                       |

Fortunately, Ethereum's specifications are executable, adding an extra layer of robustness.
Still, explicitly enumerated invariants offer compelling advantages.

First, it makes communication predictable and robust. For example, consider the following excerpt from [EIP-7623](https://eips.ethereum.org/EIPS/eip-7623):

> Any transaction with a gas limit below `21000 + TOTAL_COST_FLOOR_PER_TOKEN * tokens_in_calldata` or below its intrinsic
> gas cost (take the maximum of these two calculations) is considered invalid. This limitation exists because transactions
> must cover the floor price of their calldata without relying on the execution of the transaction.
> There are valid cases where `gasUsed` will be below this floor price, but the floor price needs to be reserved in
> the transaction gas limit.

The [testing framework](https://github.com/ethereum/execution-spec-tests) that verifies this specification across different implementations relies
on [client-specific error messages](https://github.com/ethereum/execution-spec-tests/blob/main/src/ethereum_clis/clis/geth.py) to detect violations:

{{< figure src="/blog/img/nye-proof-specs/exception-mapper.png" >}}

Even a minor typo or slight change in the client error message can lead to outcomes ranging from a [benign false positive](https://github.com/ethereum/execution-spec-tests/issues/1412) to a catastrophic security breach.
The mechanism is so abysmally delicate that you can practically hear it creaking.

An EIP-namespaced, enumerated set of invariants provides a better alternative:

| Field                 | Example                                                                                                           |
| --------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Identifier**        | `7623-01`                                                                                                         |
| **Description**       | Guarantees transactions reserve a minimum gas amount needed to cover revised calldata costs.                      |
| **Formal Expression** | `∀ Tx: gas_limit(Tx) ≥ max( intrinsic_gas_cost(Tx), 21000 + TOTAL_COST_FLOOR_PER_TOKEN × tokens_in_calldata(Tx))` |
| **Scope**             | All Ethereum transactions.                                                                                        |
| **Rationale**         | Prevents processing transactions with insufficient gas to protect against potential execution or security issues. |

The outcome is a precise, client-independent violation that is uniquely identifiable throughout the protocol:

{{< figure src="/blog/img/nye-proof-specs/better-error.png" >}}

Second, explicitly defined invariants in the specifications—and in their corresponding executable versions—allow for easier formal verification,
which can help detect conflicts as the protocol grows complex.
I'm hopeful that implementation teams will further enrich these specifications with any invariants they discover along the way. win-win.

Let's ensure that specifications leave nothing to chance, even when they're implemented on a New Year's Eve.

## References

- [Diagnosis of the OpenSSL Heartbleed Bug](https://web.archive.org/web/20141015215508/http://blog.existentialize.com/diagnosis-of-the-openssl-heartbleed-bug.html)
- [Heartbleed, Bruce Schneier](https://www.schneier.com/blog/archives/2014/04/heartbleed.html)
- [Critical crypto bug exposes Yahoo Mail, other passwords Russian roulette-style](https://arstechnica.com/information-technology/2014/04/critical-crypto-bug-exposes-yahoo-mail-passwords-russian-roulette-style/)
- [Relevant xkcd](https://xkcd.com/1354/)
