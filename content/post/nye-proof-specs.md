+++
date = '2025-04-15T19:30:32+02:00'
draft = true
title = "New Year's Eve-Proofing Your Specifications"
+++

On an unsuspecting [New Year's Eve](https://github.com/openssl/openssl/commit/4817504d069b4c5082161b02a22116ad75f822b1), the code shown below nearly crippled the Internet, resulting in damages estimated at half a billion dollars.

The code in question implements a new specification (shown on its right) for the TLS encryption protocol, aiming to solve an important bottleneck: establishing a new TLS connection is expensive. The specification proposes that the sender post an arbitrary [“heartbeat”](https://www.rfc-editor.org/rfc/rfc6520) message to the recipient. The recipient interprets this as a signal to keep the connection alive and confirms by echoing the message back to the sender. Neat. Can you spot the bug?
