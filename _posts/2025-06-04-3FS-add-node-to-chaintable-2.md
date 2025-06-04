---
layout: "post"
title: "Add a Storage Node to a Chain Table -- 2"
date: 2025-06-04
---

In the previous article, we proposed a method to add new storage nodes to a 3FS chain table. However, that method reduces data reliability of a chain because it first removes the migrating target from a chain. In this article, we propose a more secure method by adding the target to a chain first.

## Another Method to Add New Storage Nodes to a Chain Table

### Procedure Overview

1. Prepare new storage nodes. (Same as before)
1. Create new storage services. (Same as before)
1. Generate new data placement policy. (Same as before)
1. Migrate targets.

The first three steps are the same as before, so we skip them in this article. The key philosophy of this new method is: **we don't reuse existing target IDs when migrating targets in a chain; instead, we create new ones**.

### Migrate Targets

Let's start from the output of `gen_chain_table.py`, as follows:

```bash
$ docker exec 3fs-mgmtd cat /output/create_target_cmd.txt
create-target --node-id 10001 --disk-index 0 --target-id 101000100101 --chain-id 900100001  --use-new-chunk-engine
create-target --node-id 10002 --disk-index 0 --target-id 101000200101 --chain-id 900100001  --use-new-chunk-engine
create-target --node-id 10001 --disk-index 0 --target-id 101000100102 --chain-id 900100002  --use-new-chunk-engine
...
create-target --node-id 10003 --disk-index 0 --target-id 101000300131 --chain-id 900100047  --use-new-chunk-engine
create-target --node-id 10002 --disk-index 0 --target-id 101000200132 --chain-id 900100048  --use-new-chunk-engine
create-target --node-id 10003 --disk-index 0 --target-id 101000300132 --chain-id 900100048  --use-new-chunk-engine

$ docker exec 3fs-mgmtd cat /output/generated_chain_table.csv
ChainId
900100001
900100002
900100003
...
900100046
900100047
900100048

$ docker exec 3fs-mgmtd cat /output/generated_chains.csv
ChainId,TargetId,TargetId
900100001,101000100101,101000200101
900100002,101000100102,101000300101
900100003,101000100103,101000300102
...
900100046,101000200131,101000300130
900100047,101000100132,101000300131
900100048,101000200132,101000300132

$ docker exec 3fs-mgmtd cat /output/remove_target_cmd.txt
offline-target --node-id 10001 --target-id 101000100101
remove-target --node-id 10001 --target-id 101000100101
offline-target --node-id 10002 --target-id 101000200101
...

remove-target --node-id 10002 --target-id 101000200132
offline-target --node-id 10003 --target-id 101000300132
remove-target --node-id 10003 --target-id 101000300132
```

What makes this method different from the previous method is: **we don't use `target_id` from these outputs directly; we only need the `node_id` and `disk_index` of a target**.

The method works as follows:

1. Analyze every `create-target` command, record the `node_id` and `disk_index` of each chain. The key point here is we need to know the new **location of a chain (which node and which disk)**, but the ID of the target is irrelevant.
2. Process every existing chain sequentially: 
    1. Get the locations of the chain, compare current locations with new locations.
    2. If all locations of the chain do not change at all (all replicas stay in current locations), nothing needs to be done.
    3. If a location of the chain changed, we create a new target for the location with a new target ID which is different from all existing targets. Add the new target to the chain with the `update-chain --mode add` command. Wait until all targets' status becomes SERVING-UPTODATE. We do this for all changed locations first. After all new locations (new targets) become UPTODATE, we remove old targets.
3. Create new targets for new chains.
4. Generate the CSV files of chain and chain table, and upload them.

### Pros and Cons

Pros:

1. Never reduces replicas of a chain below the pre-configured value.
2. Much easier to operate. No need to process target dependencies.

Cons:

1. Consumes more capacity during target migration. This method cannot be used if free space is not enough to store a chain.

## Conclusion

In this article, we described a more secure method to add new storage nodes to a chain table. This method never reduces the number of replicas below the pre-configured value during migration, making it safer than the previous method. However, it requires more free space to store additional replicas during migration.

Although this method is easier to operate than the previous one, it is still error-prone. It is better to do this through a control plane. m3fs will release a major version to support adding new storage nodes to the cluster.
