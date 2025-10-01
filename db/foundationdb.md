# FoundationDB Transaction Logs: Architecture and Implementation

## Overview

Transaction Logs (TLogs) are a critical component of FoundationDB's distributed architecture, serving as the
Write-Ahead Log (WAL) for the entire database system. They provide ACID transaction guarantees by durably storing
all mutations before they are acknowledged to clients and making them available to storage servers for eventual
persistence.

## Core Architecture

### Purpose and Responsibilities

Transaction Logs have two primary responsibilities:
1. **Durability**: Persist incoming commits to disk and notify commit proxies when commits are durably stored
2. **Availability**: Make commits available for consumption by storage servers through a peek/pop interface

### System Position

```
Client → Commit Proxy → Transaction Log → Storage Server
                           ↓
                      Disk Storage
```

Transaction Logs sit between commit proxies (which batch client mutations) and storage servers (which provide
the key-value interface). They act as the authoritative source of truth for all committed transactions.

## Data Flow and Operations

### Commit Process

1. **Commit Proxies** collect mutations from clients into batches
2. **Tag Assignment**: Each mutation gets tagged with destination storage server identifiers
3. **TLog Commit**: Proxies send `TLogCommitRequest` containing:
   - Version information (prevVersion, version, knownCommittedVersion)
   - Serialized mutations with tags
   - Replication metadata (tLogCount, tLogLocIds)

4. **Dual Processing**: TLogs process commits in two concurrent operations:
   - **Memory Queuing**: Push mutations to in-memory per-tag queues for fast peek operations
   - **Disk Persistence**: Write to disk queue for durability and recovery

### Peek/Pop Interface

**Peek Operations** (`TLogPeekRequest`):
- Storage servers request mutations for their assigned tags
- TLogs respond with sequential mutations from in-memory queues
- Supports both blocking and non-blocking peek modes
- Can peek from specific version ranges

**Pop Operations** (`TLogPopRequest`):
- Storage servers notify TLogs when they've durably applied mutations
- Allows TLogs to free memory and disk space for processed mutations
- Essential for garbage collection and memory management

## Key Data Structures

### TLogInterface (fdbserver/include/fdbserver/TLogInterface.h)

```cpp
struct TLogInterface {
    RequestStream<TLogPeekRequest> peekMessages;
    RequestStream<TLogPeekStreamRequest> peekStreamMessages;
    RequestStream<TLogPopRequest> popMessages;
    RequestStream<TLogCommitRequest> commit;
    RequestStream<ReplyPromise<TLogLockResult>> lock;
    // ... additional control streams
};
```

### TLogData (fdbserver/TLogServer.actor.cpp:288)

The main server state structure containing:
- **SharedTLog concept**: Single process can host multiple TLog generations
- **Memory management**: Tracks bytes durable vs. bytes in memory
- **Disk components**:
  - `persistentData`: Key-value store for spilled data
  - `rawPersistentQueue`: Physical disk queue
  - `persistentQueue`: Logical queue interface

### LogData Structure

Each TLog generation has associated `LogData` containing:
- **Per-tag queues**: `TagData` structures with version-ordered messages
- **Version tracking**: Maps versions to disk locations
- **Popped version tracking**: Manages what data can be garbage collected

### TagData Structure

Per-tag data management:
```cpp
struct TagData {
    std::deque<std::pair<Version, LengthPrefixedStringRef>> versionMessages;
    Version popped;           // Latest popped version for this tag
    Version persistentPopped; // Popped version persisted to disk
    IDiskQueue::location poppedLocation; // Disk location for GC
    bool nothingPersistent;   // Optimization flag
};
```

## Memory Management and Spilling

### Memory Pressure Handling

When memory usage exceeds `TLOG_SPILL_THRESHOLD`:
1. **Spill-by-Reference**: Oldest in-memory data is moved to disk storage
2. **SQLite Backend**: Uses B-tree with key `(tag, version)` and mutation values
3. **Memory Optimization**: Maintains recent data in memory for fast peek operations

### Disk Queue Organization

**TLogQueue** (`fdbserver/TLogServer.actor.cpp:103`):
- Atomic push operations with validation flags
- Recovery support through `initializeRecovery()`
- Durable commit interface
- Packet format:
  ```
  uint32_t payloadSize
  uint8_t payload[payloadSize]
  uint8_t validFlag
  ```

## Recovery and Replication

### Recovery Process

1. **Lock Phase**: Existing TLogs are locked to prevent new commits
2. **State Recovery**: New TLogs read persistent state and queue data
3. **Generation Management**: Multiple TLog generations can coexist
4. **Handoff**: New generation takes over active commit processing

### Multi-Generation Support

TLogs support multiple generations simultaneously:
- **Active Generation**: Handles new commits
- **Recovering Generations**: Serve peek requests for historical data
- **Memory Sharing**: All generations share the same memory limits
- **Disk Queue Sharing**: Single disk queue serves all generations

### Forward Compatibility

**Log Version System** (design/tlog-forward-compatibility.md.html):
- Configurable `log_version` parameter controls on-disk format
- Supports rollback to previous versions
- Version mapping:
  - pre-5.2: log_version 1
  - 5.2-6.0: log_version 2
  - 6.1+: log_version 3
  - 6.2: log_version 4
  - 6.3: log_version 5

## Integration with Other Components

### Commit Proxies

- Send batched mutations via `TLogCommitRequest`
- Wait for durability confirmation before acknowledging clients
- Coordinate with multiple TLogs for replication

### Storage Servers

- Peek their assigned tags to get mutations to apply
- Pop TLogs after durably applying mutations
- Handle both in-memory and spilled data through unified interface

### Cluster Controller

- Manages TLog recruitment and failure detection
- Coordinates recovery processes
- Handles TLog replacement during failures

### Log System

- Orchestrates multiple TLog replicas
- Implements quorum-based durability
- Manages cross-datacenter replication

## Key Actor Functions

### Core Processing Actors

**`tLogCommit`** (TLogServer.actor.cpp):
- Main commit processing logic
- Handles both memory and disk operations
- Manages version advancement and replication

**`tLogPeekMessages`**:
- Serves peek requests from storage servers
- Handles both memory and disk-based data
- Supports streaming and batch modes

**`updatePersistentData`**:
- Manages spilling to disk
- Updates persistent popped versions
- Coordinates memory cleanup

### Recovery Actors

**Recovery coordination** in ClusterRecovery.actor.cpp:
- Handles master recruitment
- Coordinates TLog recovery
- Manages cluster state transitions

## Performance Characteristics

### Memory Usage

- Typically holds 5-7 seconds of mutations in memory
- Default 1.5GB memory limit per TLog process
- Spills to disk under memory pressure
- Overhead tracking for accurate memory management

### Disk I/O Patterns

- Sequential writes to disk queue for commits
- B-tree operations for spilled data access
- Efficient range reads for peek operations
- Batch operations to minimize disk seeks

### Scaling Properties

- Horizontal scaling through TLog sharding
- Multiple replicas for fault tolerance
- Cross-datacenter replication support
- Memory-optimized for common access patterns

## Configuration and Tuning

### Key Parameters

- `TLOG_SPILL_THRESHOLD`: Memory limit before spilling
- `TLOG_MESSAGE_BLOCK_BYTES`: Block size for disk operations
- Replication factors and quorum settings
- Cross-datacenter configuration

### Monitoring Points

- Memory usage (`bytesDurable`, `bytesInput`)
- Commit latency and throughput
- Peek/pop operation performance
- Spill frequency and disk usage

## Code Locations

### Primary Implementation
- **TLogServer.actor.cpp**: Main TLog server implementation
- **TLogInterface.h**: Interface definitions and request/reply structures
- **TLogQueue**: Disk queue abstraction

### Related Systems
- **TagPartitionedLogSystem.actor.h**: Multi-TLog coordination
- **ClusterRecovery.actor.cpp**: Recovery orchestration
- **CommitProxyServer.actor.cpp**: Commit proxy integration

### Design Documents
- **design/tlog-spilling.md.html**: Spill-by-reference design
- **design/tlog-forward-compatibility.md.html**: Version compatibility

This architecture provides FoundationDB with a robust, scalable transaction logging system that ensures ACID
properties while maintaining high performance for both read and write operations.
