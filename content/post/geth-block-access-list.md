---
date: "2025-05-26T06:07:11+02:00"
title: "Analysis of Block Access List (BAL) Using Geth"
---

> ### ⚠️ Work in progress
>
> This analysis is a work in progress. The information presented may be incorrect or subject to change.

## TL;DR

- Block-level Access Lists (BALs) enable both parallel I/O and EVM execution. This proof-of-concept focuses on I/O parallelization benefits.
- Analysis conducted on 120 blocks using hardware representative of a typical Ethereum node. Extended analysis with larger sample size planned.
- Implementation adds BALs to block headers, increasing total RLP-encoded size for 120 blocks from **15MB** to **20MB** (~**33%** overhead).
- Results show **42%** reduction in mean block processing time (**708ms** to **409ms**), dominated by **69%** improvement in storage read performance (**477ms** to **147ms**).
- Investigation required: **70%** increase in account read time and **7%** increase in execution time require investigation.

## Introduction

Block-level Access Lists (BALs) proposes a significant advancement in Ethereum's Layer 1 scalability through parallel transaction validation. While the complete BAL proposal enables both parallel I/O and EVM execution, this analysis focuses exclusively on I/O parallelization opportunities.

The core innovation allows block builders to provide explicit lists of storage slot accesses, enabling validators to parallelize state loading operations. For comprehensive details on the BAL proposal and its design considerations, refer to the [eth research post](https://ethresear.ch/t/block-level-access-lists-bals/22331).

This analysis implements a proof-of-concept BAL system in Geth focused on I/O parallelization to evaluate its potential benefits and overhead.

## Results

![Block Processing Time](/img/geth-block-access-list/block-processing.png)
_Block processing time comparison between master branch (left) and BAL (right)_

![Memory Usage](/img/geth-block-access-list/block-components.png)
_Processing time comparisons of block components between master branch(left) and BAL(right) implementation_

### Performance Metrics

| Metric (mean per block) | Master Branch | BAL Implementation | Change      |
| ----------------------- | ------------- | ------------------ | ----------- |
| Block Processing Time   | **708ms**     | **409ms**          | -**42%** ⬇️ |
| Execution Time          | **82.9ms**    | **89ms**           | +**7%** ⬆️  |
| Account Read Time       | **82.9ms**    | **141ms**          | +**70%** ⬆️ |
| Storage Read Time       | **477ms**     | **147ms**          | -**69%** ⬇️ |

### Analysis

The performance metrics reveal four key insights:

1. **Overall Performance**: Total block processing time decreased by 42%, primarily through optimized storage access patterns.

2. **Storage Access Efficiency**: Storage read time dropped by 69% (477ms to 147ms) due to parallel loading of storage slots, eliminating serial read bottlenecks during execution.

3. **Account Access Trade-offs**: The 70% increase in account read time stems from transactions competing with BAL for state access to read account information such as nonce and balance. This needs further investigations and needs to be optimized.

4. **Execution Impact**: A 7% increase in execution time warrants investigation, though it falls within expected variance ranges.

5. **Transaction Execution Pattern**:
   ![Block Execution Gantt Chart](/img/geth-block-access-list/execution-gantt.png)

   The Gantt chart visualization reveals an interesting execution pattern. Early transactions show longer execution times as they compete with BAL prefetching for state loading. However, as block execution progresses, more transactions benefit from the prefetched state, resulting in significantly faster execution times towards the tail end of the block. This demonstrates the cumulative benefit of BAL prefetching, though it suggests potential optimization opportunities for early transaction execution.

These results demonstrate BAL's potential for significant I/O parallelization benefits, even with current overhead costs.

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

For this analysis, we simulate block builder behavior by injecting BALs into existing blocks during export. In production, this would be handled by block builders during block construction.

We slightly modify the block export function `ExportN` to inject BAL into the block header. This is done by reprocessing the block which warms up the cache, the access list is then populated by reading from the cache.

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
geth export chain.bin 22551881 22552000
```

Reset the chain head from geth console:

```sh
debug.setHead("0x1581d48")

# Confirm chain head
eth.blockNumber==22551880
> true
```

Import the blocks:

```sh
geth \
        import \
        --state.scheme "path" \
        --metrics \
        --metrics.influxdb \
        --metrics.influxdb.endpoint "http://0.0.0.0:8086" \
        --metrics.influxdb.username "xxxx" \
        --metrics.influxdb.password "xxxx" \
        --metrics.influxdb.tags "host=BAL" \
        --history.logs.disable \
        --nocompaction \
        chain.bin
```

Confirm the chain head:

```sh
# Confirm import
eth.blockNumber==22552000
> true
```

### Prefetching BAL

BALs are used to prefetch state in parallel by full nodes and validators when validating a block.

This implementation extends Geth's existing prefetcher by checking for a BlockAccessList in the header. If present, it warms up accounts and storage slots in parallel. Otherwise, it falls back to the original transaction-based prefetching. All state reads are performed in parallel to maximize throughput.

```go
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

        workers.Wait()
        return
    }

    // Fall back to transaction execution based warmup if no BlockAccessList

```

## Methodology Comparison

Our implementation differs from [initial tests and simulations by EthStorage/Quarkchain](https://hackmd.io/X4Z4h-EQRPSiQF38rpN9aQ) in two key aspects:

### 1. BAL Integration Method

- **EthStorage/Quarkchain**: Imported BALs separately from JSON files during testing
- **Current Implementation**: Injects BALs directly into block headers, aligning with the BAL specification and helps estimate block size overhead.

### 2. Prefetcher Architecture

- **EthStorage/Quarkchain**: Implemented a custom BAL prefetcher
- **Current Implementation**: Extends Geth's already performant prefetcher. This helps maintain backward compatibility by falling back to existing prefetching for blocks without BALs and closely compares BALs against geth baseline.

## Measurement setup

Then analysis was conducted using a modified version of Geth (linked below) with an unmodified Lighthouse consensus client. We use a modest hardware setup, closer to an average home node.

### Hardware

- **OS**: Debian GNU/Linux 12 (bookworm)
- **Kernel**: Linux 6.1.0-32-cloud-amd64
- **Virtualization**: kvm (QEMU)
- **CPU**: 16 core vCPU @ 2.0GHz
- **Memory**: 32G
- **Go**: go1.24.2 linux/amd64

## References

1. [Block-level Access Lists (BALs)](https://ethresear.ch/t/block-level-access-lists-bals/22331) - Toni Wahrstätter, Ethereum Research
2. [BAL Implementation](https://github.com/raxhvl/go-ethereum/tree/research/block-access-list) - GitHub Repository
3. [EthStorage/Quarkchain BAL Implementation](https://hackmd.io/X4Z4h-EQRPSiQF38rpN9aQ?view) - EthStorage Team
