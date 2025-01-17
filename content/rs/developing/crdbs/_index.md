---
Title: Developing Applications with Geo-replicated CRDBs on Redis Enterprise Software (RS)
description:
weight: $weight
alwaysopen: false
categories: ["RS"]
---
Developing geo-distributed, multi-master applications can be difficult.
Application developers may have to understand a large number of race
conditions between updates to various sites, network, and cluster
failures that could reorder the events and change the outcome of the
updates performed across geo-distributed writes.

CRDBs are geo-distributed databases that span multiple RS clusters.
CRDBs depend on multi-master replication (MMR) and Conflict-free
Replicated Data Types (CRDTs) to power a simple development experience
for geo-distributed applications. CRDBs allow developers to use existing
Redis data types and commands, but understand the developers intent and
automatically handle conflicting concurrent writes to the same key
across multiple geographies. For example, developers can simply use the
INCR or INCRBY method in Redis in all instances of the geo-distributed
application, and CRDBs handle the additive nature of INCR to reflect the
correct final value. The following example displays a sequence of events
over time : t1 to t9. This CRDB has two member CRDBs : member CRDB1 and
member CRDB2. The local operations executing in each member CRDB is
listed under the member CRDB name. The "Sync" even represent the moment
where synchronization catches up to distribute all local member CRDB
updates to other participating clusters and other member CRDBs.

|  **Time** | **Member CRDB1** | **Member CRDB2** |
|  ------: | :------: | :------: |
|  t1 | INCRBY key1 7 |  |
|  t2 |  | INCRBY key1 3 |
|  t3 | GET key1<br/>7 | GET key1<br/>3 |
|  t4 | — Sync — | — Sync — |
|  t5 | GET key1<br/>10 | GET key1<br/>10 |
|  t6 | DECRBY key1 3 |  |
|  t7 |  | INCRBY key1 6 |
|  t8 | — Sync — | — Sync — |
|  t9 | GET key1<br/>13 | GET key1<br/>13 |

Databases provide various approaches to address some of these concerns

- Active-Passive Geo-distributed deployments: With active-passive
    distributions, all writes go to an active cluster. Redis Enterprise
    provides a "Replica Of" capability that provides a similar approach.
    This can be employed when the workload is heavily balanced towards
    read and very few writes. However, WAN performance and availability
    is quite flaky and traveling large distances for writes take away
    from application performance and availability.
- Two-phase Commit (2PC): This approach is designed around a protocol
    that commits a transaction across multiple transaction managers.
    Two-phase commit provides a consistent transactional write across
    regions but fails transactions unless all participating transaction
    managers are "available" at the time of the transaction. The number
    of messages exchanged and its cross-regional availability
    requirement make two-phase commit unsuitable for even moderate
    throughputs and cross-geo writes that go over WANs.
- Sync update with Quorum-based writes: This approach synchronously
    coordinates a write across majority number of replicas across
    clusters spanning multiple regions. However, just like two-phase
    commit, number of messages exchanged and its cross-regional
    availability requirement make geo-distributed quorum writes
    unsuitable for moderate throughputs and cross geo writes that go
    over WANs.
- Last-Writer-Wins (LWW) Conflict Resolution: Some systems provide
    simplistic conflict resolution for all types of writes where the
    system clocks are used to determine the winner across conflicting
    writes. LWW is lightweight and can be suitable for simpler data.
    However, LWW can be destructive to updates that are not necessarily
    conflicting. For example adding a new element to a set across two
    geographies concurrently would result in only one of these new
    elements appearing in the final result with LWW.
- MVCC (multi-version concurrency control): MVCC systems maintain
    multiple versions of data and may expose ways for applications to
    resolve conflicts. Even though MVCC system can provide a flexible
    way to resolve conflicting writes, it comes at a cost of great
    complexity in the development of a solution.

Even though types and commands in CRDBs look identical to standard Redis
types and commands, the underlying types in RS are enhanced to maintain
more metadata to create the conflict-free data type experience. This
section will detail what you need to know about developing with CRDBs on
Redis Enterprise Software.

## Compatibility

CRDBs act very much like a standard Redis database except a few
differences:

- CRDBs in this version support all major Redis data types. See the
    list of types supported in CRDBs under the Data Types
    section.
- As conflict handling rules differ between data types, some commands
    have slightly different requirements in CRDBs vs standard Redis
    databases. (ex: String type)

## Data Types

Even though the data types and methods look identical in standard Redis
and CRDBs, there are specific rules that govern the handling of
conflicting concurrent writes to each type.

From a developer's perspective, most supported datatypes work the same
as standard Redis. However, a few methods also come with specific
requirements in CRDBs.

Below is a table of the primary data types and their support levels,
followed by descriptions:

| **Data Type** | **Support Level** |
|------------|-----------------|
| Float Counters | [Supported]({{< relref "/rs/developing/crdbs/strings.md#string-data-type-with-counter-value-in-crdbs" >}}) |
| Geospatial | [Supported]({{< relref "/rs/developing/crdbs/developing-sorted-sets-crdb.md" >}}) |
| Hashes | [Supported]({{< relref "/rs/developing/crdbs/developing-hashes-crdb.md" >}}); Hash fields are treated as strings or counters |
| Integer Counters | [Supported]({{< relref "/rs/developing/crdbs/strings.md#string-data-type-with-counter-value-in-crdbs" >}}) |
| Lists | [Supported]({{< relref "/rs/developing/crdbs/developing-lists-crdb.md" >}}) |
| Sets | [Supported]({{< relref "/rs/developing/crdbs/developing-sets-crdb.md" >}}) |
| Strings | [Supported]({{< relref "/rs/developing/crdbs/strings.md" >}}) |
| Sorted Sets | [Supported]({{< relref "/rs/developing/crdbs/developing-sorted-sets-crdb.md" >}}) |
| Bitsets | Not currently supported |
| Streams | Not currently supported |

### Other Data Types

Bitmap, Bitfields, and Hyperloglog data types and operations are
not currently supported in this version of
CRDBs.

## Lua Scripts

CRDB supports Lua scripts, but unlike standard Redis, Lua scripts always
execute in effects replication mode. There is currently no way to
execute them in script-replication mode.

## Eviction

CRDBs always operate in no eviction mode. The reasoning is that if
memory is low, eviction may not help (or even worse) until garbage
collection takes place.

## Expiration

Expiration is supported with special multi-master semantics.

If a key's expiration time is changed at the same time on different
members of the CRDB, the longer extended time set via TTL on a key is
preserved. As an example:

If this command was performed on key1 on cluster #1

```src
127.0.0.1:6379> EXPIRE key1 10
```

And if this command was performed on key1 on cluster #2

```src
127.0.0.1:6379> EXPIRE key1 50
```

The EXPIRE command setting the key to 50 would win.

And if this command was performed on key1 on cluster #3:

```src
127.0.0.1:6379> PERSIST key1
```

It would win out of the three clusters hosting the CRDB as it sets the
TTL on key1 to an infinite time.

The replica responsible for the "winning" expire value is also
responsible to expire the key and propagate a DEL effect when this
happens. A "losing" replica will from this point on not be responsible
for expiring the key, unless another EXPIRE command resets the TTL.
Furthermore, a replica that is NOT the "owner" of the expired value
will:

- Silently ignore the key if a user attempts to access it in READ
    mode, e.g. treating it as if it was expired but not propagating a
    DEL.
- Expire it (sending a DEL) before making any modifications if a user
    attempts to access it in WRITE mode.

## Out-of-Memory (OOM)

If a member CRDB is in an out of memory situation, that member is marked
"inconsistent" by RS, the member stops responding to user traffic, and
the syncer initiates full reconciliation with other peers in the CRDB.

## INFO

The INFO command has an additional crdt section which provides advanced
troubleshooting information (applicable to support etc.):

|  **Section** | **Field** | **Description** |
|  ------ | ------ | ------ |
|  **CRDT Context** | crdt_config_version | Currently active CRDB configuration version. |
|   | crdt_slots | Hash slots assigned and reported by this shard. |
|   | crdt_replid | Unique Replica/Shard IDs. |
|   | crdt_clock | Clock value of local vector clock. |
|   | crdt_ovc | Locally observed CRDB vector clock. |
|  **Peers** | A list of currently connected Peer Replication peers. This is similar to the slaves list reported by Redis. |  |
|  **Backlogs** | A list of Peer Replication backlogs currently maintained. Typically in a full mesh topology only a single backlog is used for all peers, as the requested Ids are identical. |  |
|  **CRDT Stats** | crdt_sync_full | Number of inbound full synchronization processes performed. |
|   | crdt_sync_partial_ok | Number of partial (backlog based) re-synchronization processes performed. |
|   | crdt_sync_partial-err | Number of partial re-synchronization processes failed due to exhausted backlog. |
|   | crdt_merge_reqs | Number of inbound merge requests processed. |
|   | crdt_effect_reqs | Number of inbound effect requests processed. |
|   | crdt_ovc_filtered_effect_reqs | Number of inbound effect requests filtered due to old vector clock. |
|   | crdt_gc_pending | Number of elements pending garbage collection. |
|   | crdt_gc_attempted | Number of attempts to garbage collect tombstones. |
|   | crdt_gc_collected | Number of tombstones garbaged collected successfully. |
|   | crdt_gc_gvc_min | The minimal globally observed vector clock, as computed locally from all received observed clocks. |
|   | crdt_stale_released_with_merge | Indicates last stale flag transition was a result of a complete full sync. |
|  **CRDT Replicas** | A list of crdt_replica \<uid> entries, each describes the known state of a remote instance with the following fields: |  |
|   | config_version | Last configuration version reported. |
|   | shards | Number of shards. |
|   | slots | Total number of hash slots. |
|   | slot_coverage | A flag indicating remote shards provide full coverage (i.e. all shards are alive). |
|   | max_ops_lag | Number of local operations not yet observed by the least updated remote shard |
|   | min_ops_lag | Number of local operations not yet observed by the most updated remote shard |
