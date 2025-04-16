---
date: "2025-04-15T19:30:32+02:00"
draft: true
title: "New Year's Eve-Proofing Your Specifications"
---

On an unsuspecting [New Year's Eve](https://github.com/openssl/openssl/commit/4817504d069b4c5082161b02a22116ad75f822b1), the code shown below nearly crippled the Internet, resulting in damages estimated at half a billion dollars.

{{< figure src=/img/nye-proof-specs/heartbeat-code.jpg >}}

The code in question implements a new specification (shown on its right) for the [TLS encryption protocol](https://en.wikipedia.org/wiki/Transport_Layer_Security), aiming to solve an important bottleneck: establishing a new TLS connection is expensive. The specification proposes that the sender post an arbitrary [“heartbeat”](https://www.rfc-editor.org/rfc/rfc6520) message to the recipient. The recipient interprets this as a signal to keep the connection alive and confirms by echoing the message back to the sender. Neat. Can you spot the bug?

The catch is that the recipient never checks whether the supplied length actually matches the message. This allows the sender blatantly lie about the message length, tricking the recipient into leaking extra information from memory back to the sender.

The now infamous bug, cleverly named Heartbleed, lets a malicious sender peek behind the TLS curtain and expose contents from memory. In a twist of cosmic irony, it often reveals exactly the information that TLS is designed to protect—such as private keys or login credentials.

Here is an exploited memory dump with sensitive information:

Notably, the specification itself acknowledges this edge case:

How could we better nudge implementation do what it says on the tin?

More unit tests? Yes.

No deployments on New Year's Eve? A necessary sacrifice, sure.

I’d argue that the _why_ is hidden in plain sight. Throughout a specification, the structural integrity of a system—its _invariant_—is often buried in normative prose. This bug shows that it's all too easy to overlook invariants, especially on a noisy New Year's Eve.

Instead, the format below lets a specification YELL its invariants:

| Field             | Description                                                                                | Example                                                                                                           |
| ----------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Identifier        | A unique reference that makes it easy to refer to the invariant.                           | `HB-01`                                                                                                           |
| Description       | A clear, human-readable explanation of what the invariant ensures.                         | The invariant guarantees that the declared length of a heartbeat message exactly matches its actual payload size. |
| Formal Expression | A formal version of the invariant provided in mathematical notation, pseudocode, or a DSL. | ∀ heartbeat M: declared_message_length = length(M).                                                               |
| Scope             | Details where (or in which parts of the system) the invariant applies.                     | Heartbeat request and response.                                                                                   |
| Rationale         | Explains why the invariant is critical and what could go wrong if it’s violated.           | Prevents unintended side-effects caused by mismatch in the message and its declared length.                       |

## References

- [Diagnosis of the OpenSSL Heartbleed Bug](https://web.archive.org/web/20141015215508/http://blog.existentialize.com/diagnosis-of-the-openssl-heartbleed-bug.html)
- [Heartbleed, Bruce Schneier](https://www.schneier.com/blog/archives/2014/04/heartbleed.html)
