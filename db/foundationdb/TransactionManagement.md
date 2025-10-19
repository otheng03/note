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
