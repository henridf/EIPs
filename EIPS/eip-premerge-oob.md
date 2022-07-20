---
eip: tbd
title: Format for Pre-Merge Execution Data
description: Specifies file format for storage of pre-merge blocks, bodies, and receipts.
status: Draft
type: Standards Track
author: ...
category: Networking
created: 2022-07-10
---

## Abstract
A SSZ-based file storage specification for pre-Merge bodies, headers, and receipts. This format will be used to store, distribute, and verify historical data post-Merge.


## Motivation

Providing out-of-band storage, distribution, and verification of pre-Merge history is a first step towards full EIP-4444s. Some time after the Merge happens, a special-case rollout of EIP-4444 can take place, which applies exclusively to pre-Merge history.

## Specification

A block archive file consists of a header and body that are individually concatenated and encoded into a file.

```python

MAX_PAYLOADS_PER_ARCHIVE = 1000000 # tbd

class ArchiveHeader(Container):
    version: uint64
    head_block_number: uint64
    block_count: uint64

class ArchiveBody(Container):
    blocks: List[Block, MAX_PAYLOADS_PER_ARCHIVE]
```

### Block

The Block and Header containers are near-identical to the Beacon Chain's `ExecutionPayload`, with some differences due to their representing PoW blocks rather than PoS blocks. Notably, uncles and receipts are absent in the `ExecutionPayload.


##### Custom types
| Name | SSZ equivalent | Description |
| - | - | - |
| `Transaction` | `ByteList[MAX_BYTES_PER_TRANSACTION]` | either a [typed transaction envelope](https://eips.ethereum.org/EIPS/eip-2718#opaque-byte-array-rather-than-an-rlp-array) or a legacy transaction|
| `ExecutionAddress` | `Bytes20` | Address of account on the execution layer |


##### Constants
| Name | Value |
| - | - |
| `MAX_BYTES_PER_TRANSACTION` | `uint64(2**30)` (= 1,073,741,824) |
| `MAX_TRANSACTIONS_PER_PAYLOAD` | `uint64(2**20)` (= 1,048,576) |
| `MAX_UNCLES_PER_PAYLOAD` | `10` (??) |
| `BYTES_PER_LOGS_BLOOM` | `uint64(2**8)` (= 256) |
| `MAX_EXTRA_DATA_BYTES` | `2**5` (= 32) |



```python
class Block(Container):
    header: Header
    transactions: List[Transaction, MAX_TRANSACTIONS_PER_PAYLOAD]
    uncles: List[Header, MAX_UNCLES_PER_PAYLOAD]
    receipts: List[Receipt, MAX_TRANSACTIONS_PER_BLOCK]

class Header(Container):
    parent_hash: Hash32
    uncle_hash: Hash32   # not in beacon chain's ExecutionPayload
    fee_recipient: ExecutionAddress  # 'beneficiary' in the yellow paper
    state_root: Bytes32
    tx_hash: Hash32 # not in beacon chain's ExecutionPayload
    receipts_root: Bytes32
    logs_bloom: ByteVector[BYTES_PER_LOGS_BLOOM]
    prev_randao: Bytes32  # 'difficulty' in the yellow paper
    block_number: uint64  # 'number' in the yellow paper
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[MAX_EXTRA_DATA_BYTES]
    base_fee_per_gas: uint256
    mix_digest: Bytes32 # not in beacon chain's ExecutionPayload
    nonce: uint64 # not in beacon chain's ExecutionPayload
    # block_hash: Hash32   -- not sure we need this? # Hash of execution block

```

### Receipts payload

Constants
| Name | Value |
| - | - |
| `MAX_TOPICS_PER_LOG` | `uint8(2**2)` (= 4) |
| `MAX_LOGS_PER_PAYLOAD` | `uint64(2**20)` (= 1,048,576) |
| `BYTES_PER_LOGS_BLOOM` | `uint64(2**8)` (= 256) |
| `MAX_LOG_DATA_BYTES` | `uint64(2**22)` (= 4,194,304) |


```python
class Receipt(Container):
   post_state: Bytes32
   status: uint64
   cumulative_gas_used: uint64
   logs: List[Log, MAX_LOGS_PER_PAYLOAD]



class LogPayload(Container):
   address: ExecutionAddress
   topics: List[Hash32, MAX_TOPICS_PER_LOG]
   data: ByteList[MAX_LOG_DATA_BYTES]
```





## Rationale

**Use of SSZ**

We use SSZ rather than RLP, despite it being the primary encoding used in the PoW chain. One advantage of SSZ isthat it allows complete specification of structures, which can be encoded/decoded unambiguously into all languages with SSZ implementations. This makes it easy for block archive files to be used by all execution clients. Also, SSZ provides merkelization out of the box, allowing us to trivially obtain and verify roots for an entire archive as well as individual blocks and block fields.




## Reference Implementation

A reference implementation can be found at https://github.com/henridf/eip44s-proto/.

## Security Considerations

Each block archive will have a companion hash root (obtained via SSZ
`hash_tree_root()`). These are accumulated into a list (similar to the
beacon chain's `BeaconState.historical_roots`). 

After the merge, we could "seal" the accumulator and
publish its root - that root could be used to
verify any archive file (and its contents).

Of course, assuming EIP-4444 continues post merge, we will need to
start a new accumulator.

An alternate approach is not to "seal" the accumulator post-merge, but
this means we need to publish and track each file archive's root.




## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).
