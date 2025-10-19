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
