---
date: "2025-05-22T06:07:11+02:00"
title: "Analysis of Block Access List (BAL) Using Geth"
---

> ### ⚠️ Work in progress
> This analysis is a work in progress. The information presented may be incorrect or subject to change.

## TL; DR

## Introduction


## Methodology
BALs are proposed part of block headers, we use an import-export mechanism to inject BALs to block headers, and import the modified block.

```go
// 📄 core/types/block.go

// SlotAccess tracks all accesses to a specific storage slot.
type SlotAccess struct {
    Slot     common.Hash      // Storage slot being accessed
}

// AccountAccess tracks all storage accesses for an account.
type AccountAccess struct {
    Address  common.Address   // Account address
    Slots    []SlotAccess    // List of accessed storage slots
}

// Header represents a block header in the Ethereum blockchain.
type Header struct {
    ...

 	// BlockAccessList introduced by EIP-7928 and is ignored in legacy headers.
    BlockAccessList *[]AccountAccess `json:"blockAccessList" rlp:"optional"`
}
```

BAL is excluded from hash calculation to preserve original block hash.

```go
// 📄 core/types/block.go
func (h *Header) Hash() common.Hash {
	headerCopy := CopyHeader(h)
	headerCopy.BlockAccessList = nil
	return rlpHash(headerCopy)
}
```

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
