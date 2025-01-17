# Motivation

Automatic tablet splitting enables changing the number of tablets (which are splits of data) at runtime. There are a number of scenarios where this is useful:

### Range Scans
In use-cases that scan a range of data, it is often impossible to predict a good split boundary. For example: 

```
CREATE TABLE census_stats (
    age INTEGER,
    user_id INTEGER,
    ...
    );
```

In the table above, it is not possible for the database to infer the range of values for age (typically in the `1` to `100` range). It is also impossible to predict the distribution of rows in the table, meaning how many `user_id` rows will be inserted for each value of age to make an evenly distributed data split. This makes it hard to pick good split points ahead of time.

### Low-cardinality primary keys
In use-cases with a low-cardinality of the primary keys (or the secondary index), hashing is not very effective. For example, if we had a table where the primary key (or index) was the column `gender` which has only two values `Male` and `Female`, hash sharding would not be very effective. However, it is still desirable to use the entire cluster of machines to maximize serving throughput.


### Small tables that become very large
This feature is also useful for use-cases where tables begin small, and thereby start with a few shards. If these tables grow very large, then nodes continuously get added to the cluster. We may reach a scenario where the number of nodes exceeds the number of tablets. Such cases require tablet splitting to effectively re-balance the cluster.


# High-level design

## Driving tablet splitting from master side
Master is monitoring tablets and decides when to split particular tablet.
- Master configuration parameter `tablet_size_split_threshold` is propagated to all tservers inside master configuration 
data embedded into `TSHeartbeatResponsePB`.
- Tablet server reports list of tablets exceeding tablet_size_split_threshold in a `TSHeartbeatRequestPB`.
- Master sends `TabletServerAdminService.SplitTablet` RPC to leader tablet server with a list of tablets to split.
- Once tablet splitting is complete on a leader of source/old tablet - master will get info about new tablets in a 
tablet report embedded into `TSHeartbeatRequestPB`. Also we can send this info back as a response to 
`TabletServerAdminService.SplitTablet RPC`, so master knows faster about new tablets.
- Once master knows about tablet partitioning is changed it increments `SysTablesEntryPB.partition_version`.
- After leader is elected for new tablets - they are switched into `RUNNING` state.
- We keep old tablet Raft group available as a Remote bootstrap source after the split, but not available for serving 
reads/writes. This is needed, for example, in case some old tablet replica is partitioned away before split record was 
added into it’s Raft log and then it joins the cluster back after majority splits and need to bootstrap old tablet, so 
it can split. We need to keep old tablet available for follower_unavailable_considered_failed_sec seconds. After that 
timeout tserver is considered as failed and is evicted from Raft group, so we don’t need to hold old tablet anymore.

`SysTablesEntryPB.partition_version` will be included into `GetTableLocationsResponsePB`.

## Tablet splitting on tserver side
When leader tablet server receives `SplitTablet` RPC it adds a special Raft record containing:
- 2 new tablet IDs. Why it might be easier to use two new tablet ids:
  - The key range per tablet id can be aggressively cached everywhere because it never changes.
  - Handling of two new tablets' logs will be uniform, vs. separate handling for the old tablet (but with reduced key 
  range) and new tablet logs.
- Split key (chosen as the approximate mid-key). Should be encoded DocKey (or its part), so we don’t split in the middle 
of DocDB row. In case hash partitioning is used for the table - we should split by hash.
- We disallow processing any writes on the old tablet after split record is added to Raft log.

Tablet splitting happens at the moment of applying a Raft split-record and includes following steps:
- Do the RocksDB split - both regular and provisional records
- Duplicate all Raft-related objects
- Any new (received after split record is added) records/operations will have to be added directly to one of the two 
new tablets.
- Before split record is applied old tablet is continuing processing read requests in a usual way.
- Old tablet will reject processing new read/write operations after split record apply is started. Higher layers will 
have to handle this appropriately, update metadata and retry to new tablets.

## YBClient
- Currently, `YBTable::partitions_` is populated inside `YBTable::Open`. We need to refresh it on following events:
  - `MetaCache::ProcessTabletLocations` gets a newer partition_version from `GetTableLocationsResponsePB`.
  - Request to a tablet leader got rejected due to tablet has been split. As an optimization, we can include necessary 
  information about new tablets into this response for YBClient to be able to retry on new tablets without reaching to 
  master.
- Each time YBTable::partitions_ is updated we also need to update meta cache (currently done by 
`MetaCache::ProcessTabletLocations`).

**Note: To allow splitting on range key we will need to implement range partitioning first.**

## Distributed transactions
- Until old tablet is deleted, it receives “apply” requests from TransactionManager and will reject them. As a part of 
reject response it will send back new tablets IDs, so TransactionManager will retry “apply” requests to new tablets.
- When old tablet is deleted it also checks its provisional records DB for transactions in which the tablet participates 
and sends them info to update its tablet ID to new tablets IDs.

## Document Storage Layer splitting
- Copy the RocksDB to additional directory using hard links (we already have `CreateCheckpoint` function for this) and 
add  metadata saying that only part of the key range is visible. Remote bootstrap will work right away.
Next major compaction will remove all key-value pairs which are no longer related to the tablet for new tablets due to 
split. Also later we can implement cutting of RocksDB without full compaction (see below in this section).
- Store split key inside `KvStoreInfo` tablet metadata. `IntentAwareIterator` will filter out non relevant keys. We can 
also propagate key boundary to regular RocksDB instance so it has knowledge about non relevant keys and we can 
implement RocksDB optimizations like truncating SST files quickly even before the full compactions.
- **Performance note: remote bootstrap could download not relevant data from new tablet if remote bootstrap is happened 
before compaction.**

- Snapshots (created by Tablet::CreateSnapshot)
  - The simplest solution is to clone snapshot as well using hard-link, but snapshots are immutable and once they will 
  be re-distributed across different tservers it will increase space amplification. Snapshots are currently only used 
  for backup and are quickly deleted. But it would be good to implement cutting of RocksDB without full compaction - 
  could be done later. 
- Cutting of RocksDB without full compaction
  - We can “truncate” SST S-block and update SST metadata in case we cut off the first half of S-block, so RocksDB 
  positions correctly inside S-block without need to update data index.

## Provisional records DB splitting
We have the following distinct types of data in provisional records (intents) store:
- Main provisional record data:  
`SubDocKey` (no HybridTime) + `IntentType` + `HybridTime` -> `TxnId` + value of the provisional record
- Transaction metadata:  
`TxnId` -> status tablet id + isolation level
- Reverse index by Txn ID:  
`TxnId` + `HybridTime` -> Main provisional record data key

`TxnId`, `IntentType`, `HybridTime` are all prefixed with appropriate value type.

Reverse index by transaction ID is used for getting all provisional records in tablet related to particular transaction. 
We can’t just split provisional records DB at RocksDB level by some mid-key because metadata is not sorted by original 
key. Instead we can just duplicate provisional records DB and do filtering by original key inside 
`docdb::PrepareApplyIntentsBatch` which is the only function using provisional records reverse index.
Filtering of main provisional record data will be done inside IntentAwareIterator.

## Other components
We should disallow tablet splitting when the split is requested from a tablet leader that is remote-bootstrapping one 
of the nodes.
Bulk load tool relies on partitioning to be fixed during the load process, so we decided to pre-split and disable 
dynamic splitting during bulk load.
