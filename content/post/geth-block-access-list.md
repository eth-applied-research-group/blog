---
date: "2025-05-22T06:07:11+02:00"
title: "Analysis of Block Access List (BAL) Using Geth"
---

> ### ⚠️ Work in progress
> This analysis is a work in progress. The information presented may be incorrect or subject to change.

## TL; DR

## Introduction

## Methodology
BALs are proposed part of block headers, we use an import-export mechanism to inject BALs to block headers,
and import the modified block.

Export block range:

```sh
geth export chain.bin 22484036 22484045
```

Reset the chain head from geth console:

```sh
debug.setHead("0x1571443")

# Confirm chain head
eth.blockNumber==22484035
> True
```

Import the blocks:

```sh
geth import chain.bin
```

Confirm the chain head:
```sh
eth.blockNumber==22484045
```

## Measurement setup
A modified version of geth, and lighthhouse CL. We use a modest hardware setup, closer
to an average home node.

### Hardware
- **OS**: Debian GNU/Linux 12 (bookworm)
- **Kernel**: Linux 6.1.0-32-cloud-amd64
- **Virtualization**: kvm (QEMU)
- **CPU**: 16 core vCPU @ 2.0GHz
- **Memory**: 32G
- **Go**: go1.24.2 linux/amd64

## Results
