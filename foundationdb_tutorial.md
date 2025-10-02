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
