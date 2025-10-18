# Bootstrapping

```
+------------------------------------------------+
| Coordinators                                   |
| - Store configuration of LS (LogServers info)  |
| - Act as a Disk Paxos group                    |
| - Exactly one Coordinator becomes              |
|   the ClusterController                        |
+------------------------------------------------+
           |
           | (1) Elect a new ClusterController if none exists
           v
+--------------------------------------------------------+
| ClusterController                                      |
+--------------------------------------------------------+
  |                       ^                          ^
  | (2) Recruits          | (3) Reads configuration  | (7) Writes the new LS configuration to all Coordinators
  v     a new Sequencer   |     of OLD LS            |
+---------------------------------------------------------+
| A new Sequencer                                         |  (4) Spawns a new TS and LS           +------------+
|                                                         | ------------------------------------> | The new TS |
+---------------------------------------------------------+  (6) Waits until the new TS           +------------+
                                                                 finished recovery                +------------+
                                                                                                  | The new LS |             
                                                                                                  +------------+  
+------------------------------------------------+
| Proxies                                        |
| - Recover system metadata from OLD LS          |
| - Include info about all StorageServers        |
+------------------------------------------------+
           |
           | (5) Proxies recover system metadata
           |     (from OLD LS, including info about all SS)
           v
+------------------------------------------------+
| LogServers (OLD LS)                            |
| - Persist metadata about StorageServers        |
+------------------------------------------------+

+------------------------------------------------+
| StorageServers                                 |
| - Store all user data                          |
| - Store most system metadata (0xFF prefix)     |
+------------------------------------------------+

(8) At this time, the new transaction system becomes ready to accept client transactions
```
