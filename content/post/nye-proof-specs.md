+++
date = '2025-04-15T19:30:32+02:00'
draft = true
title = "New Year's Eve-Proofing Your Specifications"
+++

On an unsuspecting New Year's Eve, the code shown below nearly crippled the Internet, resulting in damages estimated at half a billion dollars.

The code in question implements a new specification (shown on its right) for the TLS encryption protocol, aiming to solve an important bottleneck: establishing a new TLS connection is expensive. The specification proposes that the sender post an arbitrary “heartbeat” message to the recipient. The recipient interprets this as a signal to keep the connection alive and confirms by echoing the message back to the sender. Neat. Can you spot the bug?
