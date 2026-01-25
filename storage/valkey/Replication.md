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

client:
```
REPLCONF listening-port 41804
REPLCONF ip-address 127.0.0.1
REPLCONF capa eof
REPLCONF capa psync2
PSYNC ? -1
```

valkey-server:
```
void syncCommand(client *c)

int startBgsaveForReplication(int mincapa, int req) {
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

# rdbSaveRio

```
⏺ RDBFLAGS Explained in Detail

  These flags control how RDB data is saved or loaded depending on the context (normal
  persistence, replication, AOF, debugging). They're bitmask flags that can be combined with |.

  Visual Overview

                           ┌─────────────────────────────────────────────┐
                           │           RDB Save/Load Contexts            │
                           └─────────────────────────────────────────────┘
                                              │
          ┌───────────────────┬───────────────┼───────────────┬──────────────────┐
          ▼                   ▼               ▼               ▼                  ▼
     ┌─────────┐      ┌──────────────┐  ┌───────────┐  ┌─────────────┐   ┌────────────┐
     │ BGSAVE  │      │ AOF Rewrite  │  │   SYNC    │  │DEBUG RELOAD │   │  Startup   │
     │(to disk)│      │  (preamble)  │  │(to replica│  │   (merge)   │   │ (from disk)│
     └────┬────┘      └──────┬───────┘  └─────┬─────┘  └──────┬──────┘   └─────┬──────┘
          │                  │                │               │                │
     RDBFLAGS_NONE    AOF_PREAMBLE      REPLICATION      ALLOW_DUP         FEED_REPL
                                       + KEEP_CACHE                       (if primary)

  ---
  Flag-by-Flag Breakdown

  RDBFLAGS_NONE (0)

  Standard RDB save to disk (BGSAVE, SAVE). No special handling.

  ---
  RDBFLAGS_AOF_PREAMBLE (1 << 0)

  When: RDB is embedded at the start of an AOF file for faster loading.

  Effects:
  - Sets aof-base AUX field to 1 (line rdb.c:1244)
  - Progress logs say "AOF rewrite" instead of "RDB" (rdb.c:1372)
  - Modules receive SUBEVENT_LOADING_AOF_START instead of LOADING_RDB_START
  - Skips deleting expired keys during load (rdb.c:3441) — AOF commands after the preamble handle
  expiration properly

  AOF file structure with preamble:
  ┌───────────────────────────┐
  │  RDB snapshot (preamble)  │  ← RDBFLAGS_AOF_PREAMBLE
  ├───────────────────────────┤
  │  AOF commands...          │
  │  SET foo bar              │
  │  INCR counter             │
  └───────────────────────────┘

  ---
  RDBFLAGS_REPLICATION (1 << 1)

  When: Full sync from primary to replica.

  Effects:
  - Includes transient cluster state like open slots (cluster_legacy.c:8066) — this data shouldn't
   persist to disk but must transfer during sync
  - Modules receive SUBEVENT_LOADING_REPL_START
  - Often combined with KEEP_CACHE since the replica will read the file immediately

  // replication.c:976 — generating RDB for replica
  rdbSaveBackground(req, server.rdb_filename, rsiptr,
                    RDBFLAGS_REPLICATION | RDBFLAGS_KEEP_CACHE);

  ---
  RDBFLAGS_ALLOW_DUP (1 << 2)

  When: DEBUG RELOAD MERGE — reload RDB without flushing existing keys.

  Effects:
  - Duplicate keys don't cause errors; they overwrite existing ones (rdb.c:3465)
  - Functions/libraries with same name are allowed (rdb.c:3055)

  Before:  DB has {foo: "old", bar: "existing"}
  RDB has: {foo: "new", baz: "added"}
  After:   DB has {foo: "new", bar: "existing", baz: "added"}

  ---
  RDBFLAGS_FEED_REPL (1 << 3)

  When: Primary loads RDB at startup while replicas are connected.

  Effects:
  - As keys load, synthesize replication commands so replicas stay in sync
  - Expired keys during load generate DEL commands to replicas (rdb.c:3442)

  // server.c:6927 — startup with connected replicas
  if (hasActiveReplicas()) {
      createReplicationBacklog();
      rdb_flags |= RDBFLAGS_FEED_REPL;  // Replicate loading to replicas
  }

  ---
  RDBFLAGS_KEEP_CACHE (1 << 4)

  When: The RDB file will be read again soon (replication, shutdown-restart).

  Effects:
  - Skips posix_fadvise(DONTNEED) which would evict file data from OS page cache
  - Normally after writing an RDB, Valkey tells the OS "I'm done with this, you can reclaim the
  memory" — this flag prevents that

  // rdb.c:1554 — during save
  if (!(rdbflags & RDBFLAGS_KEEP_CACHE))
      rioSetReclaimCache(&rdb, 1);  // Tell OS to drop pages after writing

  // replication.c:976 — keep cache for imminent replica read
  rdbSaveBackground(..., RDBFLAGS_REPLICATION | RDBFLAGS_KEEP_CACHE);

  ---
  Common Flag Combinations
  ┌───────────────────────┬────────────────────────────────────────────┐
  │       Scenario        │                   Flags                    │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ Normal BGSAVE         │ RDBFLAGS_NONE                              │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ AOF rewrite           │ RDBFLAGS_AOF_PREAMBLE                      │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ Full sync to replica  │ RDBFLAGS_REPLICATION | RDBFLAGS_KEEP_CACHE │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ Startup with replicas │ RDBFLAGS_FEED_REPL                         │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ DEBUG RELOAD MERGE    │ RDBFLAGS_ALLOW_DUP                         │
  ├───────────────────────┼────────────────────────────────────────────┤
  │ Shutdown save         │ RDBFLAGS_KEEP_CACHE (restart expected)     │
  └───────────────────────┴────────────────────────────────────────────┘
```

```C
// Sace MAGIC_STRING
snprintf(magic, sizeof(magic), "VALKEY%03d", RDB_VERSION);
if (rdbWriteRaw(rdb, magic, 9) == -1) goto werr;

// Auxiliary fields, rdbSaveInfoAuxFields
int redis_bits = (sizeof(void *) == 8) ? 64 : 32;
int aof_base = (rdbflags & RDBFLAGS_AOF_PREAMBLE) != 0;

/* Add a few fields about the state when the RDB was created. */
if (rdbSaveAuxFieldStrStr(rdb, "valkey-ver", VALKEY_VERSION) == -1) return -1;
if (rdbSaveAuxFieldStrInt(rdb, "redis-bits", redis_bits) == -1) return -1;
if (rdbSaveAuxFieldStrInt(rdb, "ctime", time(NULL)) == -1) return -1;
if (rdbSaveAuxFieldStrInt(rdb, "used-mem", zmalloc_used_memory()) == -1) return -1;

/* Handle saving options that generate aux fields. */
if (rsi) {
    if (rdbSaveAuxFieldStrInt(rdb, "repl-stream-db", rsi->repl_stream_db) == -1) return -1;
    if (rdbSaveAuxFieldStrStr(rdb, "repl-id", server.replid) == -1) return -1;
    if (rdbSaveAuxFieldStrInt(rdb, "repl-offset", server.primary_repl_offset) == -1) return -1;
}
if (rdbSaveAuxFieldStrInt(rdb, "aof-base", aof_base) == -1) return -1;

TBD

```
