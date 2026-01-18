# Run foundationdb server

```
$ docker run -d --name fdb -p 4500:4500 foundationdb/foundationdb:latest
```

# Run foundationdb client

```
$ docker exec -it fdb fdbcli
```

```
fdb> writemode on

fdb> configure new single ssd
Database created

fdb> set mykey "hello world"
Committed (4162288)

fdb> get mykey
`mykey' is `hello world'

fdb> status
Configuration:
  Redundancy mode        - single
  Storage engine         - ssd-2
  Coordinators           - 1
  Usable Regions         - 1

Cluster:
  FoundationDB processes - 1
  Zones                  - 1
  Machines               - 1
  Memory availability    - 7.7 GB per process on machine with least available
  Fault Tolerance        - 0 machines
  Server time            - 10/02/25 00:45:12

Data:
  Replication health     - Healthy
  Moving data            - 0.000 GB
  Sum of key-value sizes - 0 MB
  Disk space used        - 210 MB

Operating space:
  Storage server         - 213.7 GB free on most full server
  Log server             - 213.7 GB free on most full server

Workload:
  Read rate              - 6 Hz
  Write rate             - 0 Hz
  Transactions started   - 2 Hz
  Transactions committed - 0 Hz
  Conflict rate          - 0 Hz

Backup and DR:
  Running backups        - 0
  Running DRs            - 0

Client time: 10/02/25 00:45:12
```

## Simple Transactions

### Basic read-write transaction:


```
fdb> begin
Transaction started
fdb> set key1 "value1"
fdb> set key2 "value2"
fdb> get key1
`key1' is `value1'
fdb> commit
Committed (308194263)
```

### Transaction with rollback:


```
fdb> begin
Transaction started
fdb> set testkey "temporary"
fdb> get testkey
`testkey' is `temporary'
fdb> rollback
Transaction rolled back
fdb> get testkey
`testkey': not found
```

# How to build fdb from source on my Mac

```
$ cd <FDB_SOURCE_DIR>
$ mkdir build
$ cd build
$ cmake -G Ninja <FDB_SOURCE_DIR>
$ ninja
```

Failed to compile and replaced `cmake_minimum_required` with 3.5 in `build/toml11Project-prefix/src/toml11Project/CMakeLists.txt` as below:

```
cmake_minimum_required(VERSION 3.5)
```

[339/1190] Linking CXX shared library lib/libfdb_c.dylib
FAILED: [code=1] lib/libfdb_c.dylib 

```
cmake -G Ninja -DCMAKE_CXX_FLAGS=-Wl,-ld_classic ..
```

Retry build

```
$ cmake -G Ninja <FDB_SOURCE_DIR>
$ ninja
```
