# Replication

```C
typedef struct replBufBlock {
    int refcount;          /* Number of replicas or repl backlog using. */
    long long id;          /* The unique incremental number. */
    long long repl_offset; /* Start replication offset of the block. */
    size_t size, used;
    char buf[];
} replBufBlock;

```

The size limit of replBufBlock:

```C
/* Avoid creating nodes smaller than PROTO_REPLY_CHUNK_BYTES, so that we can append more data into them,
 * and also avoid creating nodes bigger than repl_backlog_size / 16, so that we won't have huge nodes that
 * can't trim when we only still need to hold a small portion from them. */
size_t limit = max((size_t)server.repl_backlog_size / 16, (size_t)PROTO_REPLY_CHUNK_BYTES);
size_t size = min(max(len, (size_t)PROTO_REPLY_CHUNK_BYTES), limit);
```


# CHILD INFO EVENT

```
  Flow:

  PSYNC command (syncCommand)
      ↓
  Partial resync not possible → Full sync needed
      ↓
  startBgsaveForReplication()  [replication.c:961]
      ↓
  rdbSaveToReplicasSockets()   [rdb.c:3549]
      ↓
  fork() → Child process streams RDB to replicas
      ↓
  sendChildInfoGeneric(CHILD_INFO_TYPE_REPL_OUTPUT_BYTES, ...)  [rdb.c:3656]
```

Why it happens:

1. When a replica sends PSYNC and a full resync is required (can't do partial sync), the primary needs to send the entire RDB snapshot
2. The primary forks a child process (rdbSaveToReplicasSockets) to serialize and stream the RDB data to replicas without blocking the main server
3. The child reports back to the parent via the child_info_pipe:
  - CHILD_INFO_TYPE_RDB_COW_SIZE - Copy-on-Write memory stats
  - CHILD_INFO_TYPE_REPL_OUTPUT_BYTES - How many bytes were sent (only in dual-channel mode, line 3655-3656)
4. The parent receives these stats via receiveChildInfo() to track replication progress and memory usage

Specifically for dual-channel replication (line 3655-3656): the REPL_OUTPUT_BYTES event is sent so the parent knows how much data was transferred, which gets stored in server.stat_net_repl_output_bytes for monitoring/stats purposes.


# PSYNC

options:
- repl-diskless-sync 

commands.def:
```
{MAKE_CMD("psync","An internal command used in replication.",NULL,"2.8.0",CMD_DOC_NONE,NULL,NULL,"server",COMMAND_GROUP_SERVER,PSYNC_History,0,PSYNC_Tips,0,syncCommand,-3,CMD_NO_ASYNC_LOADING|CMD_ADMIN|CMD_NO_MULTI|CMD_NOSCRIPT,ACL_CATEGORY_ADMIN|ACL_CATEGORY_DANGEROUS|ACL_CATEGORY_SLOW,NULL,PSYNC_Keyspecs,0,NULL,2),.args=PSYNC_Args},
```

void syncCommand(client *c)
  int rdbSaveToReplicasSockets(int req, rdbSaveInfo *rsi)
    if ((childpid = serverFork(CHILD_TYPE_RDB)) == 0) {
      int rdbSaveRioWithEOFMark(int req, rio *rdb, int *error, rdbSaveInfo *rsi) {
        void startSaving(int rdbflags) {
          void moduleFireServerEvent(uint64_t eid, int subid, void *data) {}
        }
        rioWrite(rdb, "$EOF:", 5)
        rioWrite(rdb, eofmark, RDB_EOF_MARK_SIZE)
        rioWrite(rdb, "\r\n", 2)
        rdbSaveRio(req, rdb, error, RDBFLAGS_REPLICATION, rsi) == C_ERR) {
          rdbWriteRaw(rdb, magic, 9)
          rdbSaveInfoAuxFields(rdb, rdbflags, rsi)
          for (j = 0; j < server.dbnum; j++) {
            rdbSaveDb(rdb, j, rdbflags, &key_counter) {
              while (kvstoreIteratorNext(kvs_it, &next)) { 
                int curr_slot = kvstoreIteratorGetCurrentHashtableIndex(kvs_it);
                  rdbSaveKeyValuePair(rdb, &key, o, expire, dbid)
              }
            }
          }
          rdbSaveType(rdb, RDB_OPCODE_EOF)
          rioWrite(rdb, &cksum, 8)
        }
        rioWrite(rdb, eofmark, RDB_EOF_MARK_SIZE)
      }
    }
```

