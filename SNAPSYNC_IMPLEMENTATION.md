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
- `ending_hash` (32 bytes) - Last account hash boundary (exclusive)
- `response_bytes` (uint64) - Soft limit for response size (64 KB - 512 KB typical)

**Request Semantics**:
- Accounts returned should be in lexicographical order of their hashes
- Hash range: [starting_hash, ending_hash)
- Responder may return partial range if byte limit is reached
- Empty range request (starting_hash >= ending_hash) is invalid

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
- All account hashes in requested range [starting_hash, ending_hash)
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
- All slot hashes within requested range [starting_hash, ending_hash)
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
  - Create continuation request: [new_start, same_ending_hash)
  - Can use same or different peer

**Why increment?** Ranges are `[start, end)` where start is inclusive. If you don't increment, you'll request the last account again, causing duplicates.

### Storage Range Requests

**Small Accounts** (estimated < 1000 slots):
- Request full range: starting_hash = 0x00...00, ending_hash = 0xff...ff
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

**Throttling Mechanism**: Uses a throttle divisor to control request rate

**Constants**:
- Base request size: 1024 nodes (maxTrieRequestCount)
- Minimum throttle: 1.0 (no throttling, request full 1024 nodes)
- Maximum throttle: 1024.0 (maximum throttling, request 1 node)
- Throttle increase factor: 1.33 (when falling behind)
- Throttle decrease factor: 1.25 (when keeping up)
- EMA impact: 0.005 (for smoothing process rate)

**Throttling Algorithm**:
```
# Measure rates
process_rate = nodes_processed / time_elapsed
arrival_rate = nodes_received / time_elapsed

# Smooth process rate with exponential moving average
ema_process_rate = 0.995 * old_ema + 0.005 * process_rate

# Adjust throttle based on whether we're keeping up
IF arrival_rate > ema_process_rate * 1.1:
    # Falling behind - INCREASE throttle (request FEWER nodes)
    throttle = throttle * 1.33
    throttle = min(maxTrienodeHealThrottle, throttle)  # cap at 1024
ELSE IF ema_process_rate > arrival_rate * 1.25:
    # Keeping up - DECREASE throttle (request MORE nodes)
    throttle = throttle / 1.25
    throttle = max(minTrienodeHealThrottle, throttle)  # floor at 1.0

# Calculate actual nodes to request
nodes_to_request = floor(maxTrieRequestCount / throttle)
# Example: throttle=2.0 → request 1024/2 = 512 nodes
```

**Why this works**: Throttle is a divisor. Higher throttle = fewer nodes requested = slower arrival rate.

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

**Main Loop Pseudocode**:
```
FUNCTION sync_phase(state_root):
    load_or_create_account_tasks()

    WHILE account_tasks_remaining():
        clean_completed_tasks()

        assign_account_requests()
        assign_bytecode_requests()
        assign_storage_requests()

        event = WAIT_FOR_EVENT([
            response_received,
            request_timeout,
            peer_joined,
            peer_dropped,
            cancellation_signal
        ])

        MATCH event:
            CASE response_received:
                validate_and_process_response(event.response)
            CASE request_timeout:
                handle_timeout(event.request)
            CASE peer_joined:
                # New peer available, try assigning tasks
                continue
            CASE peer_dropped:
                revert_peer_requests(event.peer_id)
            CASE cancellation_signal:
                RETURN cancelled

        save_progress_checkpoint()

    RETURN sync_complete
```

**Task Assignment**:
```
FUNCTION assign_account_requests():
    FOR EACH task in account_tasks:
        IF task.assigned OR task.completed:
            CONTINUE

        peer = select_best_peer(REQUEST_TYPE_ACCOUNT)
        IF peer == NULL:
            BREAK  # No available peers

        request = CREATE_REQUEST(
            id = generate_request_id(),
            state_root = current_state_root,
            starting_hash = task.next_hash,
            ending_hash = task.last_hash,
            response_bytes = calculate_byte_limit(peer)
        )

        send_get_account_range(peer, request)
        mark_task_assigned(task, request)
        mark_peer_busy(peer, REQUEST_TYPE_ACCOUNT)
        start_timeout_timer(request, peer)
```

**Response Processing**:
```
FUNCTION process_account_response(response):
    request = find_request(response.request_id)
    task = request.task

    # Validate
    IF NOT validate_account_response(response):
        reject_and_revert(response, request)
        RETURN

    # Store accounts
    FOR EACH account in response.accounts:
        write_account_to_db(account.hash, account.body)
        generate_trie_nodes(account)

        IF account.code_hash != EMPTY_CODE_HASH:
            task.pending_codes.add(account.code_hash)

        IF account.storage_root != EMPTY_STORAGE_ROOT:
            create_storage_task(account.hash, account.storage_root)

    # Check if task complete
    IF len(response.accounts) == 0:
        # Empty response means range is complete (no accounts in range)
        mark_task_completed(task)
    ELSE:
        last_received_hash = response.accounts[-1].hash

        # Check if we've reached or passed the end of the range
        IF last_received_hash >= task.last_hash:
            mark_task_completed(task)
        ELSE:
            # Partial response - need continuation
            # CRITICAL: Start next request at last_hash + 1 to avoid duplicates
            continuation_start = increment_hash(last_received_hash)
            create_continuation_task(continuation_start, task.last_hash)

    mark_peer_idle(request.peer_id, REQUEST_TYPE_ACCOUNT)
    update_progress_counters(response)
```

### Healing Phase State

**Healing Loop**:
```
FUNCTION healing_phase():
    initialize_trie_sync_scheduler(state_root)

    WHILE healing_incomplete():
        missing_nodes = trie_sync_scheduler.missing()
        missing_codes = collect_missing_bytecodes()

        IF len(missing_nodes) == 0 AND len(missing_codes) == 0:
            RETURN healing_complete

        assign_trie_node_requests(missing_nodes)
        assign_bytecode_heal_requests(missing_codes)

        event = WAIT_FOR_EVENT([
            heal_response_received,
            heal_request_timeout,
            cancellation_signal
        ])

        MATCH event:
            CASE heal_response_received:
                process_heal_response(event.response)
                update_trie_sync_scheduler(event.response)
            CASE heal_request_timeout:
                handle_heal_timeout(event.request)
            CASE cancellation_signal:
                RETURN cancelled
```

**Trie Node Healing**:
```
FUNCTION assign_trie_node_requests(missing_nodes):
    throttled_batch_size = calculate_throttle()

    FOR EACH peer in idle_peers(REQUEST_TYPE_TRIENODE):
        batch = missing_nodes.take(throttled_batch_size)
        IF len(batch) == 0:
            BREAK

        request = CREATE_REQUEST(
            id = generate_request_id(),
            state_root = current_state_root,
            paths = convert_to_path_sets(batch),
            response_bytes = 512 KB
        )

        send_get_trie_nodes(peer, request)
        track_request(request)
        start_timeout_timer(request, peer)
```

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
key = "account_path:" || path_from_root
value = rlp_encoded_node
```

*Storage trie nodes*:
```
key = "storage_path:" || account_hash || path_from_root
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

### Overview

As accounts and storage slots are downloaded, the client must reconstruct the merkle patricia trie incrementally. This is done using a "stack trie" - a memory-efficient data structure that generates trie nodes on-the-fly.

### Stack Trie Algorithm

**Concept**:
- Insert keys in sorted order
- Generate nodes as soon as possible
- Emit nodes to callback
- Keep only current "stack" in memory (O(depth) space)

**Basic Operation**:
```
FUNCTION stack_trie_insert(key, value):
    WHILE stack not empty AND key not descendant of stack.top:
        node = stack.pop()
        emit_node(node.path, node.hash, node.rlp)

    create_leaf(key, value)
    push_to_stack(leaf)
```

### Boundary Filtering (Path Scheme)

**Problem**: Nodes at range boundaries may be incomplete.

**Left Boundary**:
- First inserted item defines left boundary
- Nodes on path from root to first item are incomplete
- Must filter out left boundary nodes

**Right Boundary**:
- Last inserted item defines right boundary
- When range is incomplete, right boundary nodes are incomplete
- Must filter out right boundary nodes

**Algorithm**:
```
FUNCTION on_trie_node(path, hash, blob):
    # Left boundary filtering
    IF skip_left_boundary AND (first_path == NULL OR path is_prefix_of first_path):
        IF first_path == NULL:
            first_path = path
            # Clean up leftover nodes on left boundary
            FOR i FROM 0 TO len(path):
                delete_node_at_path(path[0:i])
        RETURN  # Skip this node

    # Extension node inner path cleanup
    IF last_path != NULL AND last_path starts_with path AND len(last_path) - len(path) > 1:
        # This is an extension node covering inner path
        FOR i FROM len(path)+1 TO len(last_path):
            delete_node_at_path(last_path[0:i])

    # Write the node
    write_node_to_database(path, blob)
    last_path = path
```

**Commit Behavior**:
```
FUNCTION commit(complete_flag):
    IF complete_flag:
        # Flush remaining nodes
        root_hash = stack_trie.hash()
        RETURN root_hash
    ELSE:
        # Discard right boundary
        FOR i FROM 0 TO len(last_path):
            delete_node_at_path(last_path[0:i])
        RETURN null
```

### Dangling Node Cleanup

**Problem**: Interrupted syncs leave "dangling" nodes in database.

**Types of Dangling Nodes**:

1. **Outer dangling nodes**: On path from root to first/last inserted item
2. **Inner dangling nodes**: Between extension node and its child

**Cleanup Strategy**:

*Left boundary cleanup* (when first node committed):
```
FOR i FROM 0 TO len(first_committed_path):
    path_prefix = first_committed_path[0:i]
    IF node_exists(path_prefix):
        delete_node(path_prefix)
```

*Right boundary cleanup* (when incomplete commit):
```
FOR i FROM 0 TO len(last_committed_path):
    path_prefix = last_committed_path[0:i]
    IF node_exists(path_prefix):
        delete_node(path_prefix)
```

*Inner path cleanup* (when extension node detected):
```
IF current_node_path is_prefix_of last_node_path:
    # Extension node detected
    FOR i FROM len(current_node_path)+1 TO len(last_node_path)-1:
        inner_path = last_node_path[0:i]
        IF node_exists(inner_path):
            delete_node(inner_path)
```

### Hash Scheme Trie Generation

Simpler approach for hash scheme:

```
FUNCTION hash_scheme_on_trie_node(path, hash, blob):
    write_node_by_hash(hash, blob)
    # No boundary filtering
    # No dangling node cleanup
```

**Commit**:
```
FUNCTION hash_scheme_commit(complete_flag):
    IF complete_flag:
        RETURN stack_trie.hash()
    ELSE:
        RETURN null
```

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
        end_hash = 0xff...ff  # All 256 bits set to 1
    ELSE:
        # Regular chunk: next chunk's start is our end (exclusive)
        end_hash = (i+1) * chunk_size

    create_task(start_hash, end_hash)
    # Task range is [start_hash, end_hash) - start inclusive, end exclusive
```

**Important**: Ranges use half-open intervals `[start, end)` where:
- `start` is inclusive (included in range)
- `end` is exclusive (NOT included in range)
- This prevents gaps and overlaps between adjacent ranges

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
    RETURN start <= hash AND hash < end
```

## Merkle Proof Verification

### Account Range Proof

**Proof Structure**:
- Array of RLP-encoded trie nodes
- Contains nodes from root to first and last accounts in range
- May contain additional nodes to prove absence of accounts

**Verification Algorithm**:
```
FUNCTION verify_account_range_proof(proof, accounts, state_root, start_hash, end_hash):
    # Build trie from proof
    trie = construct_trie_from_proof(proof)

    IF trie.root() != state_root:
        RETURN error("Proof root doesn't match state root")

    # Verify each account exists in proof
    FOR EACH account in accounts:
        IF NOT trie.can_prove(account.hash, account.body):
            RETURN error("Account not proven")

    # Verify no additional accounts exist in range
    IF len(accounts) > 0:
        last_hash = accounts[-1].hash
        IF last_hash < end_hash:
            # Proof must show no accounts exist between last_hash and end_hash
            IF NOT trie.proves_absence(last_hash + 1, end_hash):
                RETURN error("Incomplete range not proven")
    ELSE:
        # Empty range - proof must show no accounts in [start_hash, end_hash)
        IF NOT trie.proves_absence(start_hash, end_hash):
            RETURN error("Empty range not proven")

    RETURN success
```

**Proving Absence**:
```
FUNCTION proves_absence(trie, start_hash, end_hash):
    # Find the nodes that would contain accounts in this range
    left_node = trie.find_closest_node(start_hash)
    right_node = trie.find_closest_node(end_hash)

    # If they're the same node and it's proven, range is empty
    IF left_node == right_node AND left_node in proof:
        IF left_node.next_hash >= end_hash:
            RETURN true

    # Otherwise need to verify the gap
    RETURN verify_gap(trie, start_hash, end_hash)
```

### Storage Range Proof

Similar to account proof but validated against account's storage_root:

```
FUNCTION verify_storage_range_proof(proof, slots, storage_root, account_hash, start_hash, end_hash):
    trie = construct_trie_from_proof(proof)

    IF trie.root() != storage_root:
        RETURN error("Proof root doesn't match storage root")

    FOR EACH slot in slots:
        IF NOT trie.can_prove(slot.hash, slot.value):
            RETURN error("Slot not proven")

    # Similar absence proof as accounts
    IF len(slots) > 0 AND slots[-1].hash < end_hash:
        IF NOT trie.proves_absence(slots[-1].hash + 1, end_hash):
            RETURN error("Incomplete range not proven")

    RETURN success
```

## Healing Scheduler

### Trie Sync Scheduler

**Purpose**: Track missing trie nodes and schedule their retrieval.

**State**:
- Set of missing nodes (path, hash) pairs
- Set of pending requests
- Dependency graph (parent-child relationships)

**Algorithm**:
```
FUNCTION initialize_trie_sync(root_hash):
    scheduler.add_missing(path=[], hash=root_hash)

FUNCTION get_next_nodes_to_request(count):
    # Select nodes with no pending dependencies
    nodes = []
    FOR EACH (path, hash) in missing_nodes:
        IF NOT has_pending_parent(path):
            nodes.append((path, hash))
            IF len(nodes) >= count:
                BREAK
    RETURN nodes

FUNCTION on_node_received(path, hash, blob):
    # Remove from missing
    scheduler.remove_missing(path, hash)

    # Decode node and check for children
    node = decode_trie_node(blob)
    FOR EACH (child_path, child_hash) in node.children():
        IF NOT db.has_node(child_path, child_hash):
            scheduler.add_missing(child_path, child_hash)

FUNCTION pending_count():
    RETURN len(missing_nodes)
```

### Bytecode Healing

Simpler than trie healing - just a set:

```
FUNCTION initialize_bytecode_healing():
    missing_codes = set()

    # Scan all accounts for missing codes
    FOR EACH account in database:
        IF account.code_hash != EMPTY_CODE_HASH:
            IF NOT db.has_code(account.code_hash):
                missing_codes.add(account.code_hash)

    RETURN missing_codes

FUNCTION get_next_codes_to_request(count):
    RETURN missing_codes.take(count)

FUNCTION on_code_received(code_hash, code):
    missing_codes.remove(code_hash)
    db.write_code(code_hash, code)
```

## Throttling Algorithm

### Purpose

Prevent overwhelming the local node with data arriving faster than it can process.

### Rate Measurement

```
process_rate = nodes_processed / time_elapsed
arrival_rate = nodes_received / time_elapsed

# Exponential moving average for stability
ema_process_rate = 0.995 * old_ema + 0.005 * process_rate
```

### Throttle Adjustment

```
# Constants (from eth/protocols/snap/sync.go)
min_throttle = 1.0      # minTrienodeHealThrottle
max_throttle = 1024.0   # maxTrienodeHealThrottle
base_request_size = 1024  # maxTrieRequestCount

# Current throttle state (starts at 1.0, no throttling)
current_throttle = 1.0

FUNCTION update_throttle():
    IF arrival_rate > ema_process_rate * 1.1:
        # Falling behind - INCREASE throttle (request FEWER nodes)
        current_throttle = current_throttle * 1.33  # trienodeHealThrottleIncrease
        current_throttle = min(max_throttle, current_throttle)
    ELSE IF ema_process_rate > arrival_rate * 1.25:
        # Keeping up - DECREASE throttle (request MORE nodes)
        current_throttle = current_throttle / 1.25  # trienodeHealThrottleDecrease
        current_throttle = max(min_throttle, current_throttle)

FUNCTION calculate_request_size():
    throttled_size = base_request_size / current_throttle
    RETURN floor(throttled_size)
    # Example: throttle=2.0 → 1024/2 = 512 nodes
```

**Key Insight**: Throttle is a **divisor**. Higher throttle value = fewer nodes requested.

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
4. **Range Boundaries**: Ranges are `[start, end)` with exclusive end. Don't use `end-1` as it creates gaps
5. **Empty Hashes**: Check for EmptyCodeHash (0xc5d2...) and EmptyRootHash (0x56e8...) to skip unnecessary requests

### Key File References

- **Sync constants**: `eth/protocols/snap/sync.go:47-109`
- **Database prefixes**: `core/rawdb/schema.go:114-121`
- **Empty hash constants**: `core/types/hashes.go:26,32`
- **Trie generation**: `eth/protocols/snap/gentrie.go:46,294`
- **Hash partitioning**: `eth/protocols/snap/range.go:35`

---

*This document is language-agnostic and designed to facilitate implementation in any programming language. All inconsistencies have been corrected based on go-ethereum v1.14+ source code.*
