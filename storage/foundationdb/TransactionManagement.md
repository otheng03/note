Summary for "FoundationDB: A Distributed Unbundled Transactional Key Value Store"

# End-to-end Transaction Processing

A client transaction starts by contacting one of the Proxies to get a read version

Client:

```
1. Client --get a read version--> Proxy --ask a read version--> Sequencer
(The read version is guaranteed to be no less than any previously issued transaction commmit version)
2. Client --issue multiple reads and get values at that specific read version--> StorageServers
3. Client writes are buffered locally without contacting the cluster.
4. Client --sends that transaction data, including the read and write sets-->one of the Proxies
5. Client waits for a commit or abort response.
   If the transaction cannot commit. the client may choose to restart the transaction from the beginning again.
```

Proxy:
A Proxy commits a client transaction in three steps.

```
1. Proxy --get a commit version that is larger than any existing read version or commit versions--> Sequencer
   The Sequencer chooses the commit version by advancing it at a rate of one million versions per second.
2. Proxy --sends the transaction versions--> range-partitioned Resolvers, which implement an optimistic concurrency control
   If all Resolvers return with no conflict, the transaction can proceed to the final commit stage.
   Otherwise, the Proxy marks the transaction as aborted.
3. Proxy --committed transactions--> a set of LogServers (for persistence)
   A transaction is considered committed after all designated LogServers have replied to the Proxy, 
   which reports the committed version to the Sequencer (to ensure that later transactions' read versions are after this commit)
4. Proxy --reply the result of transaction --> Client
   StorageServers --continuously pull mutation logs and apply committed updates to disks--> LogServers
```

Snapshot read is supported, which helps reduce conflicts. For example, concurrent writes do not conflict with snapshot reads.

# Support Strict Serializability

FDB implements Serializable Snapshot Isolation (SSI) by combining OCC with MVCC.

Commit vresion defines a serial history for transactions and serves as Log Sequence Number.

The Sequencer returns the previous commit version (i.e., previous LSN) with commit version.

A Proxy sends both LSN and previous LSN to Resolvers and LogServers  
so that they can serially process transactions in the order of LSNs.

StorageServers pull log data from LogServers in increasing LSNs as well.

## Lock-free conflict detection algorithm on Resolvers

Each Resolver maintains a history lastCommit of recently modified key ranges by committed transactions
and their corresponding commit versions.

The commit request for Tx comprises two sets: 
- a set of modified key ranges Rw
- a set of read key ranges Rr

```
Require: lastCommit: a map of key range -> last commit version

for each range in Rr do
  ranges = lastCommit.intersect(range)
  for each r in ranges do
    if lastCommit[r] < Tx.readVersion then
      return abort
// commit path
for each range in Rw do
  lastCommit[range] = Tx.commitVersion

return commit
```

The entire key space is devided among Resolvers so that the above algorithm may be performed in parallel.

Because the modified keys expire after the MVCC window, 
the false positives are limited to only happen within the short MVCC window time (i.e., 5 seconds)

The price for OCC is to keep the recent commit history in Resolvers.

Because the lag is small (3.96ms), when client read requests reach StorageServers, 
the requested version (i.e., the latest committed data) is usually already available.
If due to a small delay the data is not available to read at a StorageServer replica,
the client either waits for the data to become available or issues a second request to another replica.
If both reads timed out, the client gets a retryable error to restart the transaction.

Because the log data is already durable on LogServers, 
StorageServers can buffer updates in memory and only persist batches of data to disks with a longer delay,
thus improving I/O efficiency by coalescing the updates.

## **Control Plane Components**
### **Coordinators**
- **`fdbserver/Coordination.actor.cpp`** - Coordination logic
- **`fdbserver/LeaderElection.actor.cpp`** - Leader election among coordinators
- **`fdbserver/CoordinatedState.actor.cpp`** - Coordinated state management

### **Cluster Controller**
- **`fdbserver/ClusterController.actor.cpp`** - Main cluster controller implementation, manages recruitment of all roles and monitors cluster health

### **Data Distributor**
- **`fdbserver/DataDistribution.actor.cpp`** - Main data distribution logic
- **`fdbserver/DDTeamCollection.actor.cpp`** - Team building and management
- **`fdbserver/DDRelocationQueue.actor.cpp`** - Manages shard movement queue
- **`fdbserver/DDShardTracker.actor.cpp`** - Tracks individual shard metrics
- **`fdbserver/DDTxnProcessor.actor.cpp`** - Transaction processing for DD
- **`fdbserver/include/fdbserver/DataDistributorInterface.h`** - Interface definition

### **Ratekeeper**
- **`fdbserver/Ratekeeper.actor.cpp`** - Rate limiting and throttling logic to prevent storage servers from being overwhelmed

## **Data Plane - Transaction System (TS) Components**
### **Proxies**
- **`fdbserver/CommitProxyServer.actor.cpp`** - Handles write transactions (path 2, 3.1, 3.2 in diagram)
- **`fdbserver/GrvProxyServer.actor.cpp`** - Handles get read version requests (path 1 in diagram)
- **`fdbserver/include/fdbserver/ProxyCommitData.actor.h`** - Shared proxy data structures

### **Sequencer** (within Master/Proxy)
The sequencer functionality is integrated into:
- **`fdbserver/masterserver.actor.cpp`** - The master server assigns commit versions (sequencing)
- **`fdbserver/CommitProxyServer.actor.cpp`** - Proxies obtain versions from master for sequencing commits

### **Resolvers**
- **`fdbserver/Resolver.actor.cpp`** - Detects transaction conflicts (path 3.2 in diagram)
- **`fdbserver/include/fdbserver/ResolverInterface.h`** - Interface definition

## **Data Plane - Log System (LS) Components**
### **LogServers (TLog)**
- **`fdbserver/TLogServer.actor.cpp`** - Main transaction log server implementation (path 2, 3.3 in diagram)
- **`fdbserver/LogSystem.cpp`** - Log system abstraction
- **`fdbserver/LogSystemConfig.cpp`** - Log system configuration
- **`fdbserver/LogSystemPeekCursor.actor.cpp`** - Reading from logs

## **Data Plane - Storage System (SS) Components**
### **StorageServers**
- **`fdbserver/storageserver.actor.cpp`** - Main storage server implementation (handles reads and writes, paths 2 and 3)
- **`fdbserver/StorageMetrics.actor.cpp`** - Storage metrics tracking
- **`fdbserver/IKeyValueStore.cpp`** - Key-value store interface
- **`fdbserver/KeyValueStoreMemory.actor.cpp`** - Memory-based storage engine
- **`fdbserver/KeyValueStoreSQLite.actor.cpp`** - SQLite storage engine
- **`fdbserver/KeyValueStoreRocksDB.actor.cpp`** - RocksDB storage engine

## **Master Server**
- **`fdbserver/masterserver.actor.cpp`** - Coordinates recovery and manages the transaction system, provides commit versions to proxies

## **Worker Process**
- **`fdbserver/worker.actor.cpp`** - Worker process that can host any role (storage, log, proxy, etc.)
- **`fdbserver/fdbserver.actor.cpp`** - Main entry point for fdbserver

The architecture follows the flow shown in your diagram: clients interact with proxies for reads and commits, proxies coordinate with resolvers for conflict detection, logs persist mutations, and storage servers provide durable storage with asynchronous replication between log and storage layers.
