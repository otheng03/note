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
      rioInitWithFd(&rdb, rdb_pipe_write);


    }
```

