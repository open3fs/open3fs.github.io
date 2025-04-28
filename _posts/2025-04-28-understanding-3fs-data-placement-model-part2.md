---
layout: post
title: "Understanding 3FS data placement model -- part 2"
date: 2025-04-28
---

Previous blog of this series: [Understanding 3FS data placement model -- part 1]({{ site.url }}/2025/04/14/understanding-3fs-data-placement-model-part1.html).

## The contents of the saved model

A saved model contains two pickle files. For example, when we generate the model with this command:

```
root@e19ee962fc32:/opt/3fs/data_placement# python3 ./src/model/data_placement.py -ql -relax -type CR --num_nodes 3 --replication_factor 2 --min_targets_per_disk 32
```

You can find these two pickle files in the model directory:

```
root@e19ee962fc32:/opt/3fs/data_placement/output/DataPlacementModel-v_3-b_48-r_32-k_2-λ_16-lb_1-ub_0# ls -l
total 4564
-rw-r--r-- 1 root root 4662465 Apr 28 01:35 data_placement.html
-rw-r--r-- 1 root root     688 Apr 28 01:35 incidence_matrix.pickle
-rw-r--r-- 1 root root     106 Apr 28 01:35 peer_traffic_map.pickle
```

Let's examine the contents of each pickle file:

**incidence_matrix.pickle** contains a mapping of node and chain relationships. The key is `(node_id, chain_id)`. If a key exists in the map, it indicates that the node serves that particular chain.

```
>>> f = open("incidence_matrix.pickle", "rb")
>>> data = pickle.load(f)
>>> print(data)
{(1, 2): True, (1, 3): True, (1, 4): True, (1, 5): True, (1, 6): True, (1, 10): True, (1, 13): True, (1, 17): True, (1, 18): True, (1, 20): True, (1, 21): True, (1, 22): True, (1, 24): True, (1, 25): True, (1, 26): True, (1, 28): True, (1, 29): True, (1, 30): True, (1, 34): True, (1, 35): True, (1, 36): True, (1, 37): True, (1, 38): True, (1, 39): True, (1, 40): True, (1, 41): True, (1, 42): True, (1, 44): True, (1, 45): True, (1, 46): True, (1, 47): True, (1, 48): True, (2, 1): True, (2, 2): True, (2, 3): True, (2, 7): True, (2, 8): True, (2, 9): True, (2, 11): True, (2, 12): True, (2, 13): True, (2, 14): True, (2, 15): True, (2, 16): True, (2, 18): True, (2, 19): True, (2, 23): True, (2, 24): True, (2, 26): True, (2, 27): True, (2, 28): True, (2, 29): True, (2, 30): True, (2, 31): True, (2, 32): True, (2, 33): True, (2, 38): True, (2, 39): True, (2, 41): True, (2, 43): True, (2, 44): True, (2, 46): True, (2, 47): True, (2, 48): True, (3, 1): True, (3, 4): True, (3, 5): True, (3, 6): True, (3, 7): True, (3, 8): True, (3, 9): True, (3, 10): True, (3, 11): True, (3, 12): True, (3, 14): True, (3, 15): True, (3, 16): True, (3, 17): True, (3, 19): True, (3, 20): True, (3, 21): True, (3, 22): True, (3, 23): True, (3, 25): True, (3, 27): True, (3, 31): True, (3, 32): True, (3, 33): True, (3, 34): True, (3, 35): True, (3, 36): True, (3, 37): True, (3, 40): True, (3, 42): True, (3, 43): True, (3, 45): True}
```

**peer_traffic_map.pickle** records the peering traffic between pairs of nodes. The key is a pair of node IDs, and the value represents the number of common chains shared between those two nodes.


```
>>> f = open("peer_traffic_map.pickle", "rb")
>>> data = pickle.load(f)
>>> print(data)
{(1, 2): 16.0, (1, 3): 16.0, (2, 1): 16.0, (2, 3): 16.0, (3, 1): 16.0, (3, 2): 16.0}
```

## RebalanceTrafficModel

### Usage

This model is used to generate new chain and chain tables when adding new nodes to a cluster. For example, if you deployed a 3-node cluster using the model like this:

```
python3 ./src/model/data_placement.py -ql -relax -type CR --num_nodes 3 --replication_factor 2 --min_targets_per_disk 32
```

And now you want to add a new node, you can generate the new model like this:

```
python3 ./src/model/data_placement.py -ql -relax -type CR --num_nodes 4 --replication_factor 2 --min_targets_per_disk 32 -m output/DataPlacementModel-v_3-b_48-r_32-k_2-λ_16-lb_1-ub_0/incidence_matrix.pickle
```

The new model will be saved to a new directory:

```
root@e19ee962fc32:/opt/3fs/data_placement/output# ls -l
total 4
drwxr-xr-x 2 root root   95 Apr 28 01:35 'DataPlacementModel-v_3-b_48-r_32-k_2-'$'\316\273''_16-lb_1-ub_0'
drwxr-xr-x 2 root root  136 Apr 28 02:29 'RebalanceTrafficModel-v_4-b_64-r_32-k_2-'$'\316\273''_11-lb_1-ub_0'
-rw-r--r-- 1 root root 3174 Apr 28 02:29  appsi_highs.log
root@e19ee962fc32:/opt/3fs/data_placement/output# cd RebalanceTrafficModel-v_4-b_64-r_32-k_2-λ_11-lb_1-ub_0/
root@e19ee962fc32:/opt/3fs/data_placement/output/RebalanceTrafficModel-v_4-b_64-r_32-k_2-λ_11-lb_1-ub_0# ls -l
total 4564
-rw-r--r-- 1 root root 4663043 Apr 28 02:29 'RebalanceTrafficModel-v_4-b_64-r_32-k_2-'$'\316\273''_11-lb_1-ub_0.html'
-rw-r--r-- 1 root root     912 Apr 28 02:29  incidence_matrix.pickle
-rw-r--r-- 1 root root     196 Apr 28 02:29  peer_traffic_map.pickle
```

Then, you can generate chain and chain table based on this new model.

### How it works

First, `RebalanceTrafficModel` reads two pieces of data from the previous *incidence_matrix.pickle*:

```
    self.existing_incidence_matrix = existing_incidence_matrix
    self.existing_disks, self.existing_groups = zip(*existing_incidence_matrix.keys())
```

The `existing_disks` is a list of node IDs (with duplicates) and `existing_groups` is a list of chain IDs (with duplicates). Both lists have a length equal to the total number of targets. They look like this:

```
self.existing_disks=(1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3)
self.existing_groups=(2, 3, 4, 5, 6, 10, 13, 17, 18, 20, 21, 22, 24, 25, 26, 28, 29, 30, 34, 35, 36, 37, 38, 39, 40, 41, 42, 44, 45, 46, 47, 48, 1, 2, 3, 7, 8, 9, 11, 12, 13, 14, 15, 16, 18, 19, 23, 24, 26, 27, 28, 29, 30, 31, 32, 33, 38, 39, 41, 43, 44, 46, 47, 48, 1, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 15, 16, 17, 19, 20, 21, 22, 23, 25, 27, 31, 32, 33, 34, 35, 36, 37, 40, 42, 43, 45)
```

Second, the model still uses the linear regression method to find a new solution, but this time it adds a new constraint and sets a new objective different from `DataPlacementModel`.

The new constraint is:

```
    def existing_targets_evenly_distributed_to_disks(model, disk):
      return po.quicksum(model.disk_used_by_group[disk,group] for group in model.groups if group <= self.num_existing_groups) <= max_existing_targets_per_disk
    model.existing_targets_evenly_distributed_to_disks_eqn = po.Constraint(model.disks, rule=existing_targets_evenly_distributed_to_disks)
```

This constraint ensures that after adding new nodes, each existing node cannot host more chains than before. This is because all targets of a node have already been assigned to a chain, leaving no spare targets.

The new objective is described as follows:

```
    # Calculate number of non-moved targets after adding new nodes
    def num_existing_targets_not_moved(model):
      return po.quicksum(model.disk_used_by_group[disk,group] for disk in model.disks for group in model.groups if (disk,group) in self.existing_incidence_matrix)

    # The value to solution: the number of targets that need to be moved
    def total_rebalance_traffic(model):
      return self.total_existing_targets - num_existing_targets_not_moved(model)

    # Objective: find the minimum value of total_rebalance_traffic
    model.obj = po.Objective(expr=total_rebalance_traffic, sense=po.minimize)
```

The objective is clear: **minimize the rebalance traffic when adding new nodes to the cluster**.
