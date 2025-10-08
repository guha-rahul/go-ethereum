# Snapshot Sync (SnapSync) Implementation Guide

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Networking Layer](#networking-layer)
4. [Database Layer](#database-layer)
5. [Algorithms](#algorithms)
6. [Testing & Validation](#testing--validation)

---

# Overview

## What is Snapshot Sync?

Snapshot sync (snap sync) is an efficient state synchronization mechanism that downloads Ethereum state data directly without executing historical transactions. It operates by:

1. **Requesting contiguous ranges** of account and storage data from peers
2. **Reconstructing the state trie** incrementally as data arrives
3. **Healing gaps** in the trie after the bulk download completes

## Why Snap Sync?

**Traditional Fast Sync Problems**:
- Downloads state trie node-by-node (inefficient)
- Many round trips for contiguous data
- Difficult to parallelize effectively

**Snap Sync Advantages**:
- Downloads raw account/storage data in ranges
- Reconstructs trie locally (fewer round trips)
- Highly parallelizable (16+ concurrent ranges)
- Resumes efficiently after interruption
- Typical mainnet sync: 1-3 hours

## Two-Phase Approach

### Phase 1: Sync Phase
Downloads the bulk of state data:
- Account ranges (addresses, balances, nonces, code hashes, storage roots)
- Storage ranges (contract storage slots)
- Bytecode (contract code)

### Phase 2: Healing Phase
Fills missing trie nodes discovered during sync:
- Trie nodes that weren't generated (at boundaries)
- Bytecode that failed to download initially

---

# Architecture

## Component Overview

### File Organization (Go-Ethereum)

**Core Sync Logic**:
- `eth/protocols/snap/sync.go` (~3200 lines) - Main coordinator, task scheduling, request/response handling
  - Constants at lines 47-109: request sizes, concurrency, throttling parameters
  - accountRequest/storageRequest/etc structs: lines 116-280
  - Syncer struct: line 444
  - Main sync loop: lines 685-771
- `eth/protocols/snap/protocol.go` - Protocol message definitions (8 message types)
- `eth/protocols/snap/handler.go` - Server-side request handlers

**Data Structures**:
- `eth/protocols/snap/gentrie.go` - Trie generation from account/storage data
  - pathTrie implementation: line 46
  - hashTrie implementation: line 294
- `eth/protocols/snap/range.go` - Hash space partitioning utilities (newHashRange: line 35)
- `core/state/snapshot/` - Snapshot data structure and management

**Database Schema**:
- `core/rawdb/schema.go` - Database key prefixes
  - Line 114: SnapshotAccountPrefix = 0x61 ("a")
  - Line 115: SnapshotStoragePrefix = 0x6f ("o")
  - Line 116: CodePrefix = 0x63 ("c")
  - Line 120: TrieNodeAccountPrefix = 0x41 ("A")
  - Line 121: TrieNodeStoragePrefix = 0x4f ("O")
  - Line 71: snapshotSyncStatusKey = "SnapshotSyncStatus"

**Constants**:
- `core/types/hashes.go` - Empty hash constants
  - Line 26: EmptyRootHash = 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421
  - Line 32: EmptyCodeHash = 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470

**Integration**:
- `eth/downloader/statesync.go` - Downloader integration wrapper
- `trie/sync.go` - State trie sync scheduler (used for healing)

## Key Data Structures

### Sync Coordinator State

The main sync coordinator tracks:
- **Database connection** - Where to store synced data
- **Storage scheme** - "hash" or "path" based trie storage
- **Current state root** - The target state root being synced
- **Account tasks** - List of account range tasks to complete
- **Healing task** - Missing nodes to fetch during healing
- **Peer pool** - Available peers and their capabilities
- **Request tracking** - Active requests per peer per message type
- **Progress counters** - Statistics on accounts/storage/bytecode synced

### Account Task

Each account task represents a chunk of the account space:
- **Range boundaries** - Start and end account hashes
- **Subtasks** - Storage tasks for accounts in this range
- **Pending requests** - Current account/bytecode/storage requests
- **Received data** - Accounts awaiting processing
- **Requirements** - Which accounts need code/storage/healing
- **Trie generator** - Local trie construction state
- **Database batch** - Buffered database writes

### Storage Task

Represents storage for one account (or sub-range of large account):
- **Account owner** - Which account this storage belongs to
- **Range boundaries** - Start and end storage slot hashes
- **Storage root** - Expected root hash of storage trie
- **Trie generator** - Storage trie construction state
- **Database batch** - Buffered writes

### Healing Task

Manages the healing phase:
- **Trie sync scheduler** - Tracks missing trie nodes
- **Pending trie nodes** - Map of paths to fetch
- **Pending bytecodes** - Set of code hashes to fetch

---

# Networking Layer

## Protocol Specification

### Basic Information

- **Protocol Name**: `snap`
- **Protocol Version**: 1 (SNAP/1)
- **Transport**: devp2p RLPx
- **Maximum Message Size**: 10 MB
- **Message Count**: 8 (4 request/response pairs)

### Message Type Identifiers

```
0x00 - GetAccountRange (request)
0x01 - AccountRange (response)
0x02 - GetStorageRanges (request)
0x03 - StorageRanges (response)
0x04 - GetByteCodes (request)
0x05 - ByteCodes (response)
0x06 - GetTrieNodes (request)
0x07 - TrieNodes (response)
```

## Message Specifications

### 1. GetAccountRange / AccountRange

**Purpose**: Retrieve a contiguous range of accounts from the state trie.

#### GetAccountRange Request (0x00)

**Fields**:
- `request_id` (uint64) - Unique identifier to match response
- `state_root` (32 bytes) - Root hash of target state trie
- `starting_hash` (32 bytes) - First account hash to include (inclusive)
- `ending_hash` (32 bytes) - Last account hash to include (INCLUSIVE)
- `response_bytes` (uint64) - Soft limit for response size (64 KB - 512 KB typical)

**Request Semantics**:
- Accounts returned should be in lexicographical order of their hashes
- Hash range: [starting_hash, ending_hash] - **BOTH ends inclusive**
- Responder may return partial range if byte limit is reached
- Empty range request (starting_hash > ending_hash) is invalid
- Handler implementation: `eth/protocols/snap/handler.go:317` - breaks when `hash >= Limit` AFTER adding account

#### AccountRange Response (0x01)

**Fields**:
- `request_id` (uint64) - Matches request identifier
- `accounts` (array) - List of account data:
  - `account_hash` (32 bytes) - Keccak256 hash of account address
  - `account_body` (bytes) - RLP-encoded slim account data
    - Slim format: [nonce, balance, storage_root, code_hash]
    - Nonce: uint64
    - Balance: big integer (uint256)
    - Storage root: 32-byte hash
    - Code hash: 32-byte keccak256 hash
- `proof` (array of byte arrays) - Merkle proof validating the range

**Response Semantics**:
- Accounts must be sorted by hash
- Proof must validate the returned range against state_root
- Empty accounts array with proof is valid (indicates no accounts in range)
- Partial response: last account becomes new starting_hash for continuation

**Proof Structure**:
- Contains all trie nodes from root to first account
- Contains all trie nodes from root to last account (or range end)
- Proves no additional accounts exist beyond last returned account up to ending_hash

### 2. GetStorageRanges / StorageRanges

**Purpose**: Retrieve storage slots for one or more accounts.

#### GetStorageRanges Request (0x02)

**Fields**:
- `request_id` (uint64) - Unique request identifier
- `state_root` (32 bytes) - State trie root for validation
- `account_hashes` (array of 32-byte hashes) - Accounts whose storage to retrieve
- `starting_hash` (32 bytes) - First storage slot hash (for large contracts)
- `ending_hash` (32 bytes) - Last storage slot boundary (for large contracts)
- `response_bytes` (uint64) - Soft response size limit

**Request Semantics**:
- Can request storage for multiple accounts in single message
- For complete storage retrieval: starting_hash = 0x00...00, ending_hash = 0xff...ff
- For large contract partial retrieval: specify sub-range with starting/ending_hash
- Responder returns storage for as many accounts as fit in byte limit

#### StorageRanges Response (0x03)

**Fields**:
- `request_id` (uint64) - Matches request ID
- `slots` (array of arrays) - Storage data per account:
  - One array per requested account (in request order)
  - Each array contains storage slots:
    - `slot_hash` (32 bytes) - Keccak256 hash of storage key
    - `slot_value` (bytes) - RLP-encoded storage value (uint256)
- `proof` (array of byte arrays) - Merkle proof for last incomplete account

**Response Semantics**:
- Slots must be sorted by hash within each account's array
- Empty slot array means account has no storage in requested range
- Proof only required if last account's storage is incomplete
- Missing accounts represented as empty arrays

**Proof Structure**:
- Validates storage range for the last account only
- Verifies against that account's storage_root
- Proves completeness of returned slots up to ending_hash

### 3. GetByteCodes / ByteCodes

**Purpose**: Retrieve contract bytecode by code hash.

#### GetByteCodes Request (0x04)

**Fields**:
- `request_id` (uint64) - Request identifier
- `code_hashes` (array of 32-byte hashes) - Keccak256 hashes of desired code
- `response_bytes` (uint64) - Soft size limit

**Request Semantics**:
- Batch multiple bytecode requests (typically 20-100)
- Responder returns as many as fit in byte limit
- Order should be preserved when possible

#### ByteCodes Response (0x05)

**Fields**:
- `request_id` (uint64) - Matches request
- `codes` (array of byte arrays) - Raw bytecode blobs

**Response Semantics**:
- Each code's keccak256 must match corresponding requested hash
- Missing codes represented as empty entries in array
- Order should match request order
- Empty/null entries indicate unavailable code

### 4. GetTrieNodes / TrieNodes

**Purpose**: Request specific trie nodes by path (for healing phase).

#### GetTrieNodes Request (0x06)

**Fields**:
- `request_id` (uint64) - Request identifier
- `state_root` (32 bytes) - State trie root
- `paths` (array of path sets) - Node locations:
  - Each path set is an array of byte arrays
  - Format depends on node type (see below)
- `response_bytes` (uint64) - Soft size limit

**Path Encoding**:

*For account trie node*:
```
path_set = [[account_path]]
```

*For storage trie nodes*:
```
path_set = [[account_path], [storage_path_1], [storage_path_2], ...]
```

- `account_path`: Nibble-encoded path from account trie root to node
- `storage_path`: Nibble-encoded path from storage trie root to node
- Paths represent the route through the trie, not the key

#### TrieNodes Response (0x07)

**Fields**:
- `request_id` (uint64) - Matches request
- `nodes` (array of byte arrays) - Raw trie node RLP blobs

**Response Semantics**:
- Nodes must be in same order as paths in request
- Missing nodes represented as empty/null entries
- Each node must be valid RLP-encoded trie node
- Node at position i corresponds to path set i

## Request/Response Flow

### Basic Synchronization Flow

```
CLIENT                                    PEER
  |                                        |
  |-- GetAccountRange(range 0x00-0x0f) -->|
  |                                        |
  |                                   [look up accounts]
  |                                   [generate proof]
  |                                        |
  |<---- AccountRange(100 accounts) ------|
  |                                        |
[verify proof]                             |
[store accounts]                           |
[identify: 80 need storage, 20 need code]  |
  |                                        |
  |-- GetByteCodes(20 code hashes) ------>|
  |-- GetStorageRanges(80 accounts) ----->|
  |                                        |
  |<---- ByteCodes(20 codes) -------------|
  |<---- StorageRanges(80 storages) ------|
  |                                        |
[process and store]                        |
[continue with next account range]         |
```

### Parallel Request Pattern

Maximize throughput by parallelizing across peers and ranges:

```
PEER A: GetAccountRange(0x00-0x0f) -----> AccountRange ----> Process
PEER B: GetAccountRange(0x10-0x1f) -----> AccountRange ----> Process
PEER C: GetAccountRange(0x20-0x2f) -----> AccountRange ----> Process
PEER A: GetStorageRanges(batch 1) ------> StorageRanges --> Process
PEER B: GetStorageRanges(batch 2) ------> StorageRanges --> Process
...
```

**Parallelization Guidelines**:
- Default: 16 concurrent account ranges
- Each account range can be assigned to different peer
- Storage requests can be parallelized independently
- Don't duplicate requests for same data
- Track which peer has which request

## Peer Management

### Peer State Tracking

For each connected peer, maintain:

**Capabilities**:
- Supported protocols (check for SNAP/1)
- Maximum message size
- Connection quality

**Activity State**:
- Active request count per message type
- Idle/busy status per message type (account, storage, bytecode, trie node)
- Last activity timestamp

**Performance Metrics**:
- Throughput (bytes per second) per message type
- Average round-trip time
- Response success rate
- Timeout count

**Reliability Flags**:
- "Stateless" flag (peer failed to deliver valid state data)
- Consecutive failure count
- Last failure reason

### Peer Selection Algorithm

When assigning a new request:

**Step 1: Filter Eligible Peers**
- Must support SNAP/1
- Not marked as "stateless"
- Idle for the specific request type needed
- Below maximum concurrent request limit
- Not recently timed out

**Step 2: Calculate Capacity Score**
```
capacity_score = throughput_bytes_per_sec / (1 + current_active_requests)
```
- Higher score = prefer this peer
- Balances throughput with current load

**Step 3: Select and Assign**
- Choose peer with highest capacity score
- Generate unique request_id
- Send request message
- Mark peer as busy for this request type
- Start timeout timer
- Record request metadata

### Timeout Handling

Each request requires timeout management:

**Timeout Calculation**:
```
timeout_duration = max(
    MIN_TIMEOUT (2 seconds),
    min(
        MAX_TIMEOUT (60 seconds),
        2 * peer_round_trip_time + size_estimate / peer_throughput
    )
)
```

**On Timeout**:
1. Remove request from active requests map
2. Mark peer as idle for that request type
3. Decrement peer's reliability score
4. Re-queue request for retry with different peer
5. If repeated timeouts: consider marking peer as "slow"

**On Peer Disconnect**:
1. Iterate all active requests for this peer
2. Revert each request (unassign from task)
3. Re-queue all reverted requests
4. Remove peer from peer pool
5. Trigger reassignment cycle

## Response Validation

### Account Range Validation

**Phase 1: Structural Validation**
- Response ID matches request ID
- Accounts array size reasonable (< maximum)
- All account hashes in requested range [starting_hash, ending_hash] (inclusive)
- Accounts sorted in strict lexicographical order by hash
- No duplicate account hashes
- Each account body is valid RLP
- Account body decodes to [nonce, balance, storage_root, code_hash]

**Phase 2: Cryptographic Validation**
- Proof is array of valid RLP-encoded trie nodes
- Reconstruct subtrie from proof nodes
- Subtrie root must equal requested state_root
- All returned accounts must exist in proof
- Proof must demonstrate no additional accounts exist between last account and ending_hash

**Phase 3: Data Validation**
- Nonce is valid uint64
- Balance is valid uint256 (no leading zeros in RLP)
- Storage root is exactly 32 bytes
- Code hash is exactly 32 bytes

**Rejection Actions**:
- Invalid proof → Mark peer as "stateless", close connection, retry with different peer
- Accounts out of range → Reject response, revert request, consider peer unreliable
- Malformed RLP → Reject response, revert request
- Unsorted accounts → Reject response, consider peer broken

### Storage Range Validation

**Phase 1: Structural Validation**
- Response ID matches request
- Number of slot arrays equals number of requested accounts
- All slot hashes within requested range [starting_hash, ending_hash] (inclusive)
- Slots sorted by hash within each account
- No duplicate slot hashes within account
- Each slot value is valid RLP

**Phase 2: Cryptographic Validation**
- If proof provided: validate against account's storage_root
- Proof must demonstrate completeness of returned slots
- Reconstructed storage trie must match storage_root

**Phase 3: Data Validation**
- Each slot value decodes to valid uint256
- Slot hash matches keccak256 of original storage key

**Rejection Actions**:
- Similar to account validation
- Invalid storage proof → Mark peer as stateless
- Mismatched storage_root → Major error, peer likely malicious

### Bytecode Validation

**Validation Steps**:
1. Response ID matches request ID
2. Code count ≤ requested count
3. For each code:
   - If not empty: keccak256(code) must match requested hash
   - If empty: acceptable (peer doesn't have this code)

**Rejection Actions**:
- Hash mismatch → Mark peer as "stateless"
- Too many codes → Reject response
- Invalid structure → Reject response

### Trie Node Validation

**Validation Steps**:
1. Response ID matches request
2. Node count matches path count
3. Each node is valid RLP-encoded trie node
4. Node structure matches expected type (branch/extension/leaf)
5. Node hash matches expected hash at that path

**Rejection Actions**:
- Invalid node RLP → Reject, revert request
- Hash mismatch → Mark peer as "stateless"
- Wrong node count → Reject response

## Request Sizing Strategy

### Account Range Requests

**Sizing Philosophy**:
- Soft limits (peer can return less, should not return more)
- Balance roundtrips vs overhead
- Adapt to peer capacity

**Typical Sizes**:
- Fast peers: 512 KB requests
- Medium peers: 256 KB requests
- Slow peers: 64 KB requests

**Adaptive Strategy**:
```
IF peer consistently returns full byte limit:
    response_bytes = min(512 KB, response_bytes * 1.2)
ELSE IF peer returns < 50% of limit:
    response_bytes = max(64 KB, response_bytes * 0.8)
```

**Continuation Handling**:
- If response contains accounts but is incomplete:
  - **CRITICAL**: Next starting_hash = increment(last_returned_account_hash)
  - Create continuation request: [new_start, same_ending_hash]
  - Can use same or different peer

**Why increment?** Ranges are `[start, end]` with both ends inclusive. If you don't increment, you'll request the last account again, causing duplicates.

### Storage Range Requests

**Small Accounts** (estimated < 1000 slots):
- Request full range: starting_hash = 0x00...00, ending_hash = 0xff...ff (covers all slots)
- Batch multiple small accounts in single request (up to 10-20 accounts)

**Large Accounts** (estimated > 1000 slots):
- Split into 16 sub-ranges
- Request each sub-range independently
- Parallelize across multiple peers
- Each sub-range: 64-512 KB limit

**Batching Strategy**:
```
accumulated_size = 0
accounts_in_batch = []

FOR EACH account needing storage:
    estimated_storage_size = estimate_from_account_info()

    IF accumulated_size + estimated_storage_size > byte_limit:
        SEND GetStorageRanges(accounts_in_batch)
        accounts_in_batch = [account]
        accumulated_size = estimated_storage_size
    ELSE:
        accounts_in_batch.append(account)
        accumulated_size += estimated_storage_size
```

### Bytecode Requests

**Typical Batch Size**: 20-100 code hashes per request

**Estimation Logic**:
- Most contracts < 24 KB
- Maximum contract size: 24 KB (protocol limit)
- Request 4x more hashes than would fit if all were maximum size
- Typical request: 85 code hashes (assuming 512 KB limit)

**Priority**:
- Code requests block account processing
- Higher priority than storage requests
- Batch codes from multiple account ranges

### Trie Node Requests (Healing)

**See detailed Throttling Algorithm section below** for complete throttle logic.

## Error Handling

### Network-Level Errors

**Connection Loss**:
```
ON peer_disconnect(peer_id):
    active_requests = get_requests_for_peer(peer_id)
    FOR EACH request in active_requests:
        revert_request(request)
        requeue_for_retry(request)
    remove_peer(peer_id)
```

**Timeout**:
```
ON request_timeout(request):
    mark_peer_idle(request.peer_id, request.type)
    reduce_peer_score(request.peer_id)
    revert_request(request)
    requeue_for_retry(request)

    IF peer_timeout_count > THRESHOLD:
        mark_peer_as_slow(request.peer_id)
```

**Malformed Message**:
```
ON malformed_message(peer_id, message):
    log_error("Malformed message", peer_id, message_type)
    close_peer_connection(peer_id)
    revert_all_requests(peer_id)
```

### Protocol-Level Errors

**Invalid Proof**:
```
ON invalid_proof(peer_id, response):
    log_warning("Invalid proof from peer", peer_id)
    mark_peer_stateless(peer_id)
    reject_response(response)
    revert_request(response.request_id)
    requeue_for_retry(response.request_id)

    IF peer_invalid_proof_count > THRESHOLD:
        disconnect_peer(peer_id)
```

**Data Hash Mismatch**:
```
ON hash_mismatch(peer_id, expected, actual):
    log_warning("Hash mismatch", peer_id, expected, actual)
    mark_peer_stateless(peer_id)
    reject_data(data)
    revert_request(request)
    requeue_for_retry(request)
```

**Incomplete Data (Not an Error)**:
```
ON partial_response(response):
    # This is acceptable - peer might have limits
    accept_partial_data(response)
    create_continuation_request(response.last_item)
    assign_continuation_to_peer()
```

## Synchronization State Machine

### Sync Phase State

**Main Loop**:

**Implementation**: See `Sync()` in `eth/protocols/snap/sync.go:585-771`

**Structure**:
```
1. Load or create 16 account range tasks (lines 775-889)
2. Loop until all tasks complete (line 685):
   a. Clean completed tasks (lines 687-688): cleanAccountTasks(), cleanStorageTasks()
   b. Exit if tasks empty AND healing done (line 689)
   c. Assign requests to idle peers (lines 691-704)
   d. Wait for events via select{} (lines 705-768)
   e. Process responses or handle failures
3. Enter healing phase when snapped=true
```

**Key Functions**:
- `assignAccountTasks()` - line 1020
- `assignStorageTasks()` - line 1220
- `assignBytecodeTasks()` - line 1140
- `cleanAccountTasks()` - line 956
- `processAccountResponse()` - line 1885

**Task Assignment**:

**Implementation**: See `assignAccountTasks()` in `eth/protocols/snap/sync.go:1020-1114`

**Algorithm**:
1. Sort idle peers by capacity (lines 1025-1040)
2. For each task without active request (line 1045: `task.req == nil && task.res == nil`)
3. Match task to highest-capacity idle peer
4. Create request with:
   - `origin = task.Next` (line 1082) - **CRITICAL**: advances on each response
   - `limit = task.Last` (line 1083)
   - `bytes = min(peer_capacity, maxRequestSize)` (line 1099)
5. Send GetAccountRange request (line 1109)
6. Set timeout based on peer RTT (lines 1086-1090)

**Note**: Completed tasks (`task.done == true`) are removed by `cleanAccountTasks()` before assignment.

**Response Processing**:

**Implementation**: See `processAccountResponse()` in `eth/protocols/snap/sync.go:1885-2490`

**Key Concepts**:
1. **Overflow trimming** (lines 1890-1907): Trim accounts beyond `task.Last`
2. **Requirement tracking** (lines 1910-1968): Identify which accounts need code/storage
3. **Storage task creation** (lines 2030-2208): Create storage subtasks for large contracts
4. **Account storage** (lines 2440-2462): Write accounts to database, generate trie nodes
5. **Task advancement** (lines 2465-2477):
   - `task.Next` advances via `incHash()` as accounts complete
   - `task.done = !res.cont` marks task completion
   - Completed tasks removed by `cleanAccountTasks()`

**Critical**: Tasks are **NOT recreated** for continuation. The same task is reassigned with `task.Next` as the new origin.

### Healing Phase State

**Healing Phase**:

**Implementation**: Integrated into main sync loop after `snapped=true`

**Trie Node Healing**: See `assignTrienodeHealTasks()` in `eth/protocols/snap/sync.go:1377-1477`
- Uses `trie.Sync` scheduler to track missing nodes (line 1379)
- Throttles request size based on processing rate (line 1409)
- Converts node requests to path sets (lines 1460-1473)
- See Throttling Algorithm section for throttle details

**Bytecode Healing**: See `assignBytecodeHealTasks()` in `eth/protocols/snap/sync.go:1505-1583`
- Scans for missing bytecodes (via `trie.Sync`)
- Batches code hash requests

**Completion**: Loop exits when `len(tasks)==0 && healer.scheduler.Pending()==0` (line 689)

---

# Database Layer

## Storage Architecture

### Storage Schemes

Ethereum clients support two trie storage schemes:

#### Hash Scheme (Legacy)

**Key Structure**:
```
key = keccak256(rlp_encoded_node)
value = rlp_encoded_node
```

**Properties**:
- Nodes referenced by hash
- Content-addressed storage
- Simpler implementation
- Higher storage overhead (duplicate nodes)
- No path information retained

#### Path Scheme (Modern)

**Key Structure**:

*Account trie nodes*:
```
key = 0x41 || hex_path
     ("A" prefix + variable hex-encoded path from root)
value = rlp_encoded_node
```

*Storage trie nodes*:
```
key = 0x4f || account_hash || hex_path
     ("O" prefix + 32 bytes account hash + variable hex path)
value = rlp_encoded_node
```

**Properties**:
- Nodes referenced by path
- Deduplication possible
- Lower storage overhead
- Supports efficient state diffs
- Requires path tracking

### Database Schema

#### Account Data

**Raw Account Storage** (Snapshot layer):
```
key = 0x61 || account_hash
     ("a" prefix + 32 bytes hash = 33 bytes total)
value = RLP([nonce, balance, storage_root, code_hash])
```

Note: 0x61 = ASCII 'a' = SnapshotAccountPrefix

**Account Trie Nodes** (Path scheme):
```
key = 0x41 || hex_path
     ("A" prefix + variable length hex-encoded path)
value = RLP(node) where node is branch/extension/leaf
```

Note: 0x41 = ASCII 'A' = TrieNodeAccountPrefix

#### Storage Data

**Raw Storage Slots** (Snapshot layer):
```
key = 0x6f || account_hash || slot_hash
     ("o" prefix + 32 bytes + 32 bytes = 65 bytes total)
value = RLP(slot_value) (uint256)
```

Note: 0x6f = ASCII 'o' = SnapshotStoragePrefix

**Storage Trie Nodes** (Path scheme):
```
key = 0x4f || account_hash || hex_path
     ("O" prefix + 32 bytes + variable hex path)
value = RLP(node)
```

Note: 0x4f = ASCII 'O' = TrieNodeStoragePrefix

#### Bytecode

**Contract Code**:
```
key = 0x63 || code_hash
     ("c" prefix + 32 bytes hash = 33 bytes total)
value = raw_bytecode (bytes)
```

Note: 0x63 = ASCII 'c' = CodePrefix

#### Sync Progress

**Sync Status**:
```
key = "SnapshotSyncStatus" (fixed string key)
value = JSON({
    account_tasks: [...],
    accounts_synced: uint64,
    accounts_bytes: uint64,
    bytecodes_synced: uint64,
    storage_synced: uint64,
    ...
})
```

## Trie Generation

**Implementation**: `eth/protocols/snap/gentrie.go`

### Overview

As accounts and storage slots are downloaded, the client must reconstruct the merkle patricia trie incrementally using a "stack trie" - a memory-efficient data structure that generates trie nodes on-the-fly.

**Key Properties**:
- Inserts keys in sorted order
- Generates nodes as soon as possible
- Keeps only current "stack" in memory (O(depth) space)
- Emits nodes via callback for immediate persistence

### Path Scheme Trie Generation

**Implementation**: `pathTrie` struct in `gentrie.go:46-293`

**Constructor**: `newPathTrie()` at line 61
- Parameters: owner hash, skipLeftBoundary flag, database reader, write batch
- Used for both account tries and storage tries

**Key Methods**:
- `update()` (line 69): Insert key-value pair into trie
- `commit()` (line 93): Finalize trie, returns root hash or nil
- `onTrieNode()` (line 115): Callback for each generated node

**Boundary Filtering** (lines 115-190):
- **Problem**: Range boundaries may have incomplete nodes
- **Left boundary**: Skips nodes on path to first inserted item (lines 120-146)
- **Right boundary**: Deletes nodes on incomplete commit (lines 99-106)
- **Inner path cleanup**: Removes dangling nodes between extension nodes (lines 152-168)

**Dangling Node Cleanup**:
- Left boundary cleanup: lines 125-145 (delete prefixes of first path)
- Right boundary cleanup: lines 99-106 (delete prefixes of last path)
- Inner path cleanup: lines 152-168 (delete inner extension node paths)

### Hash Scheme Trie Generation

**Implementation**: `hashTrie` struct in `gentrie.go:295-334`

**Constructor**: `newHashTrie()` at line 300
- Simpler than path scheme - no boundary handling needed

**Key Difference**:
- No boundary filtering (nodes referenced by hash, not path)
- No dangling node cleanup required
- Direct write to database using node hash as key

**Methods**:
- `update()` (line 304): Insert key-value pair
- `commit()` (line 308): Returns root hash, no cleanup needed

## Database Operations

### Batch Writing

**Motivation**: Reduce database write overhead by batching.

**Batch Management**:
```
batch_size_threshold = 8 MB

FUNCTION maybe_flush_batch(batch):
    IF batch.size() >= batch_size_threshold:
        batch.write()
        batch.reset()
```

**Per-Task Batches**:
- Each account task has its own batch
- Each storage task has its own batch
- Batches flushed independently
- Final flush on task completion

### Write Operations

**Account Write** (Snapshot layer):
```
FUNCTION write_account(account_hash, account_data):
    batch.put(0x61 || account_hash, account_data)  # "a" prefix
    maybe_flush_batch(batch)
```

**Storage Write** (Snapshot layer):
```
FUNCTION write_storage_slot(account_hash, slot_hash, slot_value):
    batch.put(0x6f || account_hash || slot_hash, slot_value)  # "o" prefix
    maybe_flush_batch(batch)
```

**Trie Node Write (Path Scheme)**:
```
FUNCTION write_account_trie_node(path, blob):
    batch.put(0x41 || path, blob)  # "A" prefix
    maybe_flush_batch(batch)

FUNCTION write_storage_trie_node(account_hash, path, blob):
    batch.put(0x4f || account_hash || path, blob)  # "O" prefix
    maybe_flush_batch(batch)
```

**Trie Node Write (Hash Scheme)**:
```
FUNCTION write_trie_node_by_hash(hash, blob):
    # Hash scheme stores by keccak256(node) as key
    batch.put(hash, blob)
    maybe_flush_batch(batch)
```

**Bytecode Write**:
```
FUNCTION write_bytecode(code_hash, code):
    batch.put(0x63 || code_hash, code)  # "c" prefix
    maybe_flush_batch(batch)
```

### Delete Operations

**Node Deletion (Path Scheme)**:
```
FUNCTION delete_account_node(path):
    key = 0x41 || path  # "A" prefix
    IF exists(key):
        batch.delete(key)

FUNCTION delete_storage_node(account_hash, path):
    key = 0x4f || account_hash || path  # "O" prefix
    IF exists(key):
        batch.delete(key)
```

### Read Operations

**Account Read** (Snapshot layer):
```
FUNCTION read_account(account_hash):
    data = db.get(0x61 || account_hash)  # "a" prefix
    IF data == NULL:
        RETURN NULL
    RETURN decode_account(data)
```

**Node Existence Check** (Path scheme):
```
FUNCTION has_account_node(path):
    RETURN db.has(0x41 || path)  # "A" prefix

FUNCTION has_storage_node(account_hash, path):
    RETURN db.has(0x4f || account_hash || path)  # "O" prefix
```

## Progress Persistence

### Checkpoint Data

**What to Save**:
```
sync_status = {
    "account_tasks": [
        {
            "next_hash": "0x...",
            "last_hash": "0x...",
            "storage_tasks": [
                {
                    "account": "0x...",
                    "next_hash": "0x...",
                    "last_hash": "0x...",
                    "root": "0x..."
                },
                ...
            ],
            "storage_completed": ["0x...", ...]
        },
        ...
    ],
    "accounts_synced": 1234567,
    "accounts_bytes": 98765432,
    "bytecodes_synced": 45678,
    "bytecodes_bytes": 12345678,
    "storage_synced": 9876543,
    "storage_bytes": 345678901,
    "trienode_heal_synced": 4567,
    "trienode_heal_bytes": 234567,
    "bytecode_heal_synced": 123,
    "bytecode_heal_bytes": 45678
}
```

### Save Frequency

**When to Save**:
- Every N accounts synced (e.g., 10,000)
- Every M seconds (e.g., 30 seconds)
- On graceful shutdown
- After completing major task (account range)

**Save Procedure**:
```
FUNCTION save_progress():
    FOR EACH task in account_tasks:
        task.trie_generator.commit(complete=false)  # Flush partial nodes
        task.batch.write()  # Flush database writes

    status = serialize_sync_status()
    db.put("SnapshotSyncStatus", status)
```

### Resume Logic

**On Startup**:
```
FUNCTION load_progress():
    status_data = db.get("SnapshotSyncStatus")
    IF status_data == NULL:
        RETURN create_fresh_tasks()

    status = deserialize(status_data)

    FOR EACH task_data in status.account_tasks:
        task = restore_account_task(task_data)

        # Recreate trie generator
        task.trie_generator = create_trie_generator(
            skip_left_boundary = (task.next_hash != 0x00...00)
        )

        # Recreate storage tasks
        FOR EACH storage_data in task_data.storage_tasks:
            storage_task = restore_storage_task(storage_data)
            task.storage_tasks.add(storage_task)

    RETURN status
```

## Database Integrity

### Verification

**Account Trie Verification**:
```
FUNCTION verify_account_trie(expected_root):
    iterator = create_account_iterator()
    computed_root = compute_trie_root_from_iterator(iterator)

    IF computed_root != expected_root:
        RETURN error("Account trie root mismatch")

    RETURN success
```

**Storage Trie Verification**:
```
FUNCTION verify_storage_trie(account_hash, expected_storage_root):
    iterator = create_storage_iterator(account_hash)
    computed_root = compute_trie_root_from_iterator(iterator)

    IF computed_root != expected_storage_root:
        RETURN error("Storage trie root mismatch")

    RETURN success
```

### Repair Strategies

**On Corruption Detection**:
1. Identify corrupted account/storage ranges
2. Mark tasks as incomplete
3. Delete corrupted data
4. Re-request from peers

**On Dangling Node Detection**:
1. Scan for unreferenced nodes
2. Attempt to rebuild path references
3. If unable: delete and re-sync range

---

# Algorithms

## Hash Space Partitioning

### Overview

The account hash space (2^256 possible hashes) must be divided into manageable chunks for parallel retrieval.

### Partitioning Algorithm

**Goal**: Create N equal-sized ranges

**Calculation**:
```
total_space = 2^256
chunk_count = N (typically 16)
chunk_size = floor(total_space / chunk_count)

FOR i FROM 0 TO chunk_count-1:
    start_hash = i * chunk_size

    IF i == chunk_count-1:
        # Last chunk: use max hash to cover everything
        end_hash = 0xff...ff  # All 256 bits set to 1 (MaxHash)
    ELSE:
        # Regular chunk: next chunk's start minus 1 is our end (inclusive)
        end_hash = (i+1) * chunk_size - 1

    create_task(start_hash, end_hash)
    # Task range is [start_hash, end_hash] - BOTH inclusive
    # See: hashRange.End() in eth/protocols/snap/range.go:66-72
```

**Important**: Ranges use **closed intervals** `[start, end]` where:
- BOTH `start` and `end` are inclusive (both included in range)
- Adjacent ranges are contiguous: range[i].end + 1 = range[i+1].start
- Last range has end = MaxHash (0xff...ff)

### Range Operations

**Increment Hash**:
```
FUNCTION increment_hash(hash):
    # Treat hash as 256-bit big-endian integer
    value = big_int(hash)
    value = value + 1
    IF value >= 2^256:
        RETURN 0x00...00  # Wrap around
    RETURN hash(value)
```

**Hash Range Check**:
```
FUNCTION hash_in_range(hash, start, end):
    RETURN start <= hash AND hash <= end
```

## Merkle Proof Verification

**Implementation**: `VerifyRangeProof()` in `trie/proof.go:478`

### Account Range Proof

**See**: `trie/proof.go:478-542` for complete verification algorithm

**Key Verification Steps**:
1. **Root verification**: Reconstruct trie from proof nodes, verify root matches state root
2. **Key presence**: All returned accounts must be provable in the reconstructed trie
3. **Range completeness**: Proof must demonstrate no accounts exist between last returned account and range end
4. **Edge cases**: Empty ranges must prove absence of any accounts in range

**Usage in snapsync**: Called in `processAccountResponse()` at `eth/protocols/snap/sync.go:1908`

### Storage Range Proof

**See**: Same `VerifyRangeProof()` function, but verified against account's `storage_root`

**Differences from account proof**:
- Root is account's `storage_root` instead of state root
- Keys are storage slot hashes
- Values are RLP-encoded slot values

**Usage in snapsync**: Called in `processStorageResponse()` at `eth/protocols/snap/sync.go:2068`

## Healing Scheduler

### Trie Sync Scheduler

**Implementation**: Uses `trie.Sync` from `trie/sync.go`

**Purpose**: Track missing trie nodes and schedule their retrieval using dependency-aware scheduling.

**Initialization**: See `trie.NewSync()` in `trie/sync.go:74`
- Starts with root node as missing
- Builds dependency graph as nodes are processed
- Tracks pending requests and missing nodes

**Key Methods**:
- `AddSubTrie()` - Add a trie root to track (line 89)
- `AddCodeEntry()` - Add bytecode to track (line 99)
- `Missing()` - Get count of missing nodes (line 195)
- `Commit()` - Persist processed nodes to database (line 297)

**Usage in snapsync**:
- Initialized in `healTask` at `eth/protocols/snap/sync.go:450`
- Accessed via `healer.scheduler` throughout healing phase
- Node retrieval in `assignTrienodeHealTasks()` at line 1379

### Bytecode Healing

**Implementation**: Integrated into same `trie.Sync` scheduler

**Tracking**: See `trie.Sync.AddCodeEntry()` in `trie/sync.go:99`
- Maintains set of missing code hashes
- Simpler than trie healing (no dependencies)
- Just needs to fetch and verify hash matches

**Usage in snapsync**: `assignBytecodeHealTasks()` at `eth/protocols/snap/sync.go:1505-1583`

## Throttling Algorithm

### Purpose

Prevent overwhelming the local node with trie node data arriving faster than it can process during healing phase.

### Implementation

**See**: `processTrienodeHealResponse()` in `eth/protocols/snap/sync.go:2320-2366`

**Constants** (lines 74-95):
- `trienodeHealRateMeasurementImpact = 0.005` - EMA smoothing factor
- `minTrienodeHealThrottle = 1` - Minimum throttle divisor
- `maxTrienodeHealThrottle = 1024` - Maximum throttle divisor
- `trienodeHealThrottleIncrease = 1.33` - Multiplier when overloaded
- `trienodeHealThrottleDecrease = 1.25` - Divisor when keeping up
- `maxTrieRequestCount = 1024` - Base request size

**Initial State** (line 540):
- Throttle starts at `maxTrienodeHealThrottle` (1024.0) - conservative start

**Rate Tracking** (lines 2320-2347):
- Uses exponential moving average (EMA) of processing rate
- Formula: `HR(N) = (1-MI)^N*(OR-NR) + NR` (geometric sequence)
- Tracks `trienodeHealRate` - nodes processed per second

**Throttle Adjustment** (lines 2349-2365):
- Runs every 1 second
- If `pending_nodes > 2 * processing_rate`: multiply throttle by 1.33 (request fewer)
- Otherwise: divide throttle by 1.25 (request more)
- Clamp between 1 and 1024

**Request Sizing** (line 1450 in `assignTrienodeHealTasks()`):
- `request_size = maxTrieRequestCount / throttle`
- Example: throttle=2.0 → 1024/2 = 512 nodes
- Higher throttle = fewer nodes requested = less load

**Key Insight**: Throttle is a **divisor**. Higher value = smaller requests.

---

# Testing & Validation

## Unit Testing

### Hash Space Partitioning Tests

**Test Cases**:
1. Equal division into N chunks
2. Complete coverage (no gaps)
3. No overlaps between chunks
4. Boundary cases (chunk 0 starts at 0x00..00, last chunk ends at 0xff..ff)
5. Non-divisible space (e.g., 2^256 / 17)

**Example**:
```
TEST partition_hash_space():
    ranges = create_account_ranges(chunk_count=16)

    ASSERT len(ranges) == 16
    ASSERT ranges[0].start == 0x00..00
    ASSERT ranges[15].end == 0xff..ff

    FOR i FROM 0 TO 14:
        ASSERT ranges[i].end + 1 == ranges[i+1].start  # No gaps
```

### Trie Generation Tests

**Test Cases**:
1. Empty trie (no accounts)
2. Single account
3. Multiple accounts in order
4. Boundary filtering (left/right)
5. Dangling node cleanup
6. Extension node inner path cleanup

**Example**:
```
TEST trie_boundary_filtering():
    generator = create_path_trie(skip_left_boundary=true)

    # Insert accounts
    generator.update(key1, value1)
    generator.update(key2, value2)
    generator.update(key3, value3)

    # Commit as incomplete (right boundary)
    generator.commit(complete=false)

    # Check that boundary nodes were filtered
    ASSERT NOT db.has_node(path_to_first_node)
    ASSERT NOT db.has_node(path_to_last_node)

    # Check that internal nodes were written
    ASSERT db.has_node(path_to_internal_node)
```

### Proof Verification Tests

**Test Cases**:
1. Valid proof with accounts
2. Valid proof with empty range
3. Invalid proof (wrong root)
4. Invalid proof (missing accounts)
5. Incomplete proof (doesn't cover range)
6. Malicious proof (extra accounts)

**Example**:
```
TEST verify_account_proof():
    # Create valid proof
    accounts = [account1, account2, account3]
    proof = generate_valid_proof(accounts, state_root)

    result = verify_account_range_proof(
        proof, accounts, state_root,
        start_hash=accounts[0].hash,
        end_hash=accounts[-1].hash + 1
    )

    ASSERT result == success

    # Test with invalid root
    wrong_root = hash("wrong")
    result = verify_account_range_proof(
        proof, accounts, wrong_root,
        start_hash=accounts[0].hash,
        end_hash=accounts[-1].hash + 1
    )

    ASSERT result == error
```

## Integration Testing

### Sync Against Test Chain

**Setup**:
1. Create small test chain (1000 blocks)
2. Run full node to generate state
3. Test client syncs against full node

**Validation**:
```
TEST full_sync_integration():
    full_node = start_geth_full_node()
    test_client = start_test_client()

    sync_result = test_client.snap_sync(full_node.state_root)

    ASSERT sync_result == success
    ASSERT test_client.account_count == full_node.account_count
    ASSERT test_client.state_root == full_node.state_root

    # Verify random accounts
    FOR i FROM 0 TO 100:
        random_account = full_node.get_random_account()
        client_data = test_client.get_account(random_account.hash)
        ASSERT client_data == random_account
```

### Resume After Interruption

**Test**:
```
TEST resume_interrupted_sync():
    test_client = start_test_client()

    # Start sync
    sync_task = test_client.start_snap_sync(state_root)

    # Let it sync partially
    wait(30 seconds)

    # Interrupt
    test_client.kill()
    progress = test_client.get_sync_progress()
    ASSERT progress.accounts_synced > 0
    ASSERT progress.accounts_synced < total_accounts

    # Resume
    test_client = start_test_client()
    sync_result = test_client.snap_sync(state_root)

    ASSERT sync_result == success
    # Should resume from checkpoint, not restart
    ASSERT test_client.state_root == state_root
```

### Healing Phase Tests

**Test**:
```
TEST healing_fills_gaps():
    # Create state with intentional gaps
    test_client = start_test_client()
    test_client.load_incomplete_state()  # Missing trie nodes

    healing_result = test_client.run_healing_phase()

    ASSERT healing_result == success
    ASSERT test_client.has_complete_trie()

    # Verify state is valid
    computed_root = test_client.compute_state_root()
    ASSERT computed_root == expected_root
```

## Adversarial Testing

### Invalid Proof Handling

**Test**:
```
TEST reject_invalid_proofs():
    malicious_peer = create_fake_peer()
    test_client = start_test_client()

    # Send invalid proof
    malicious_peer.send_account_range_with_wrong_proof()

    # Client should reject
    ASSERT test_client.rejected_response()
    ASSERT malicious_peer in test_client.stateless_peers
```

### Hash Mismatch Handling

**Test**:
```
TEST reject_hash_mismatches():
    malicious_peer = create_fake_peer()
    test_client = start_test_client()

    # Send bytecode with wrong hash
    malicious_peer.send_bytecode_with_wrong_content()

    ASSERT test_client.rejected_response()
    ASSERT malicious_peer in test_client.stateless_peers
```

### Resource Exhaustion

**Test**:
```
TEST throttle_prevents_overload():
    fast_peers = create_multiple_fast_peers(count=20)
    test_client = start_test_client(
        max_memory=256 MB,
        max_pending_nodes=10000
    )

    test_client.snap_sync_with_throttling()

    # Should not exceed resource limits
    ASSERT test_client.memory_usage < 256 MB
    ASSERT test_client.pending_nodes < 10000
    ASSERT sync_completed successfully
```

## Performance Testing

### Benchmark Metrics

**Measure**:
- Time to sync mainnet state
- Bandwidth consumption
- Peak memory usage
- Database write throughput
- Healing phase duration

**Example**:
```
TEST benchmark_mainnet_sync():
    peers = connect_to_mainnet_peers(count=50)
    test_client = start_test_client()

    start_time = now()
    sync_result = test_client.snap_sync(mainnet_state_root)
    end_time = now()

    ASSERT sync_result == success

    PRINT "Sync time:", end_time - start_time
    PRINT "Accounts synced:", test_client.accounts_synced
    PRINT "Bandwidth used:", test_client.bytes_downloaded
    PRINT "Peak memory:", test_client.peak_memory_usage
    PRINT "Healing time:", test_client.healing_duration
```

---

## Implementation Roadmap

### Phase 1: Core Networking
1. Implement SNAP/1 protocol messages
2. Implement request/response handlers
3. Implement peer management
4. Test against geth peer

### Phase 2: Database Layer
1. Choose storage scheme (path recommended)
2. Implement database schema
3. Implement batch writing
4. Implement trie generation with boundary filtering

### Phase 3: Sync Logic
1. Implement hash space partitioning
2. Implement account range sync
3. Implement storage range sync
4. Implement bytecode sync
5. Implement progress persistence

### Phase 4: Healing
1. Implement trie sync scheduler
2. Implement trie node healing
3. Implement bytecode healing
4. Implement throttling

### Phase 5: Testing & Optimization
1. Unit tests for all components
2. Integration tests
3. Performance optimization
4. Mainnet testing

---

## References

- **EIP-2464**: Snap Sync Protocol Specification
- **devp2p**: Ethereum Wire Protocol
- **Geth Implementation**: `eth/protocols/snap/` directory
- **Merkle Patricia Trie**: Ethereum Yellow Paper Appendix D

---

## Critical Implementation Notes

### Common Mistakes to Avoid

1. **Database Key Prefixes**: Use single-byte prefixes (0x61, 0x6f, 0x41, 0x4f, 0x63), NOT string prefixes like "account:"
2. **Continuation Logic**: Always increment the last hash when creating continuation requests to avoid requesting duplicates
3. **Throttling**: Throttle is a DIVISOR. Increase throttle to request fewer nodes, decrease to request more
4. **Range Boundaries**: Ranges are `[start, end]` with BOTH ends inclusive. Adjacent ranges are: range[i].end+1 = range[i+1].start
5. **Empty Hashes**: Check for EmptyCodeHash (0xc5d2...) and EmptyRootHash (0x56e8...) to skip unnecessary requests

### Key File References

- **Sync constants**: `eth/protocols/snap/sync.go:47-109`
- **Database prefixes**: `core/rawdb/schema.go:114-121`
- **Empty hash constants**: `core/types/hashes.go:26,32`
- **Trie generation**: `eth/protocols/snap/gentrie.go:46,294`
- **Hash partitioning**: `eth/protocols/snap/range.go:35`

---

*This document is language-agnostic and designed to facilitate implementation in any programming language. All inconsistencies have been corrected based on go-ethereum v1.14+ source code.*
