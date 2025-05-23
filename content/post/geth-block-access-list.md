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

### BAL Data structure
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
### BAL header generation

We slightly modify the block export function `ExportN` to inject BAL into the block header. This is done by reprocessing the block which warms up the cache, theaccess list is then populated by reading from the cache. 

```go
// 📄 core/blockchain.go

func (bc *BlockChain) ExportN(w io.Writer, first uint64, last uint64) error {
    ...

	// Reprocess the block to warm up state cache.
	parentBlock := bc.GetHeader(block.ParentHash(), block.NumberU64()-1)
	statedb, err := state.New(parentBlock.Root, bc.statedb)	
	if err != nil {
		return err
	}

	_, err = bc.processor.Process(block, statedb, bc.vmConfig)
	if err != nil {
	  return err
	}

	// Store the access list in the block header.
	accessList := statedb.GetBlockAccessList()
	block.SetBlockAccessList(accessList)
}
```

The `GetBlockAccessList` method populates the data from the cache.

```go
// 📄 core/state/statedb.go

// GetBlockAccessList returns the block access list from the
// state cache.
func (s *StateDB) GetBlockAccessList() *[]types.AccountAccess {
	var accesses []types.AccountAccess
	
	// Collect storage slot accesses for each accessed account
	for account, state := range s.stateObjects {
		// Get all storage slots accessed for this account
		slots := make([]types.SlotAccess, 0)
		for slot := range state.originStorage {
			slots = append(slots, types.SlotAccess{
				Slot: slot,
			})
		}

		// Only add account if there were storage accesses
		if len(slots) > 0 {
			accesses = append(accesses, types.AccountAccess{
				Address: account,
				Slots:   slots,
			})
		}
	}

	return &accesses
}
```

BAL is excluded from hash calculation to preserve original block hash. This allows for verification of block import.

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
### Prefetching BAL
The code below extends the alredy performant prefetcher which warms up the state caches 
jIt first checks for a BlockAccessList in the header - if present, it directly warms up those accounts and storages
in parallel. Otherwise, it falls back to executing transactions to discover
the state access patterns. All reads are performed in parallel to maximize
throughput.

```
// 📄 core/state_prefetcher.go

	// If block has a BlockAccessList, use it directly for warming up the cache
	if block.Header().BlockAccessList != nil {
		// Process BlockAccessList accounts in parallel
		for _, access := range *block.Header().BlockAccessList {
			workers.Go(func() error {
				// If block prefetch was interrupted, abort
				if interrupt != nil && interrupt.Load() {
					return nil
				}

				// Load account data
				reader.Account(access.Address)

				// Load all storage slots for this account
				for _, slot := range access.Slots {
					if interrupt != nil && interrupt.Load() {
						return nil
					}
					reader.Storage(access.Address, slot.Slot)
				}
				return nil
			})
		}

		// In parallel, warm up the transaction recipients and their code
		for _, tx := range block.Transactions() {
			workers.Go(func() error {
				if interrupt != nil && interrupt.Load() {
					return nil
				}

				// Preload the sender
				sender, err := types.Sender(signer, tx)
				if err == nil {
					reader.Account(sender)
				}

				// Preload the recipient and its code if it exists
				if tx.To() != nil {
					account, _ := reader.Account(*tx.To())

					// Preload the contract code if the destination has non-empty code
					if account != nil && !bytes.Equal(account.CodeHash, types.EmptyCodeHash.Bytes()) {
						reader.Code(*tx.To(), common.BytesToHash(account.CodeHash))
					}
				}
				return nil
			})
		}

		workers.Wait()
		return
	}

	// Fall back to transaction execution based warmup if no BlockAccessList
    
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
