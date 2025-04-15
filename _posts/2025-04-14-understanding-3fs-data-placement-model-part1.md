---
layout: post
title: "Understanding 3FS data placement model -- part 1"
date: 2025-04-14
---

3FS provides a script to calculate data placement strategy of a cluster: <https://github.com/deepseek-ai/3FS/blob/main/deploy/data_placement/README.md>.

This blog post will guide you through the fundamental concepts and procedures of this script, helping you understand how it works and why it's designed the way it is.

## Overview

The script approaches data placement strategy calculation as a linear programming problem. It defines two key variables that need to be solved under a set of carefully designed constraints.

## Key Concepts

Firstly, here are some notable things you should understand before digging any further:

1. **Node vs Disk Terminology**: 
   - The script treats each storage node as a virtual disk
   - Parameters `num_targets_per_disk` and `min_targets_per_disk` actually refer to per storage node
   - For clarity, we'll use "per node" instead of "per disk" throughout this article

2. **Chain vs Group Terminology**:
   - The script uses "group" to refer to what we commonly call a "chain"
   - "Number of groups" = "Number of chains"
   - "Group size" = "Chain size"

3. **Group Size**:
   - Represents the number of replicas in a group/chain
   - Determines the replication factor for data placement

4. **Qlinearize Parameter**:
   - The `--qlinearize` argument controls how strictly the script considers whether two nodes can be placed in the same chain
   - Enabling this parameter makes the placement more conservative

## Standard Configuration

The script supports various configuration options, but let's examine a typical setup:

```python
chain_table_type="CR"    # Chain replication type
bibd_only=False          # Whether to use balanced incomplete block design
qlinearize=True          # Enable strict chain placement rules
relax_lb=1               # Lower bound relaxation for recovery traffic
relax_ub=0               # Upper bound relaxation for recovery traffic
```

With `all_targets_used = True`, the total number of targets equals `number_of_chains * chain_replica_size`. For example, a setup with 100 chains and a replica size of 3 requires 300 targets.

## Find params

This is a function calculating the number of targets on each node needed by a setup, and also returns the number of chains to be created.

```python
# v = num_nodes
# k = group_size = replicator_size
# min_r = min_targets_per_disk (min targets of each node)
# max_r = 100 (if not specified, each node can have 100 targets at most)
# bibd_only = bibd_only
@staticmethod
def find_params(v, k, min_r=1, max_r=100, bibd_only=False):
  if bibd_only: min_r = max(min_r, k)
  for r in range(min_r, max_r):
    # v * r % k == 0: means number of targets of each node must be an integer multiple of replica size.
    # r * (k - 1) >= v - 1: means if a node is down, remaining active number of targets
    #                       must be >= remaining number of nodes. That means each remaining node
    #                       should host at least one target.
    # b = v * r // k: b means the number of targets in each replica.
    #                 it is the number of chain to be created.
    if v * r % k == 0 and r * (k - 1) >= v - 1:
      b = v * r // k
      if not bibd_only or r * (k - 1) % (v - 1) == 0:
        return v, b, r, k
  raise ValueError(f"cannot find valid params: {v=}, {k=}")
```

## The variables to be solved

The script solves for two main binary variables:

```python
model.disk_used_by_group = po.Var(model.disks, model.groups, domain=po.Binary)
```

This variable means for each node in the cluster, whether the node belongs to a chain.

```python
model.disk_in_same_group = po.Var(model.disk_pairs, model.groups, domain=po.Binary)
```

This variable means for each node pair, if the two nodes of the pair belong to a specific chain.

## Constraints

### Constraint 1: Relation between chain and node

The script defines some constraints to define the mathematical relation between node and chain.

```python
# If two nodes in same chain, the equation must be true. That means all the var in the equation
# must equal to 1.
def define_disk_in_same_group_lower_bound(model, disk, peer, group):
  return model.disk_used_by_group[disk,group] + model.disk_used_by_group[peer,group] <= model.disk_in_same_group[disk,peer,group] + 1

# The following constraints says: if two nodes are in the same chain, they must each be in that chain.

def define_disk_in_same_group_upper_bound1(model, disk, peer, group):
  return model.disk_in_same_group[disk,peer,group] <= model.disk_used_by_group[disk,group]

def define_disk_in_same_group_upper_bound2(model, disk, peer, group):
  return model.disk_in_same_group[disk,peer,group] <= model.disk_used_by_group[peer,group]
```

### Constraint 2: Relation between chain and target

```python
def each_disk_has_limited_capcity(model, disk):
  if self.all_targets_used:
    return po.quicksum(model.disk_used_by_group[disk,group] for group in model.groups) == self.num_targets_per_disk
  else:
    return po.quicksum(model.disk_used_by_group[disk,group] for group in model.groups) <= self.num_targets_per_disk
model.each_disk_has_limited_capcity_eqn = po.Constraint(model.disks, rule=each_disk_has_limited_capcity)
```

Mentioned above: consider `all_targets_used` is `True` only.

This constraint means: the number of chains that one node can join equals to the number of targets on the node.

### Constraint 3: Chain's replicator size and nodes

```python
def enough_disks_assigned_to_each_group(model, group):
  return po.quicksum(model.disk_used_by_group[disk,group] for disk in model.disks) == self.group_size

model.enough_disks_assigned_to_each_group_eqn = po.Constraint(model.groups, rule=enough_disks_assigned_to_each_group)
```

This constraint means: the number of nodes in a chain must equal to the replicator size of the chain.

### Constraint 4: Distribute chains evenly

The script is trying to distribute chains evenly across the cluster by applying constraints on recovery traffic properties.

```python
@property
def recovery_traffic_factor(self):
  # If the chain type is CR (replicas), only need to recovery data from one healthy node.
  return (self.group_size - 1) if self.chain_table_type == "EC" else 1

@property
def sum_recovery_traffic_per_failure(self):
  # If a node is outage, how many targets must be recovered.
  return self.num_targets_per_disk * self.recovery_traffic_factor

@property
def max_recovery_traffic_on_peer(self):
  # If a node is outage, what is the max number of targets need to be recovered from a peer node?
  # If chains distributed evenly across the cluster, each target on a node will belong to a different
  # chain and each chain contains different peers.
  # E.g. For a replica = 2 and num_nodes = 17 setup. One node has 16 targets, they belong to 16 chains,
  # and each chain contains different peers (16 different peer nodes).
  # If the node is outage, we will need to recover data from all these 16 peer nodes. And each
  # peer node is responsible for one target's recovery data.
  # If replica = 2 and num_nodes = 9, this value is 2.
  return math.ceil(self.sum_recovery_traffic_per_failure / (self.num_nodes-1))
```

```python
def calc_peer_recovery_traffic(model, disk, peer):
  if self.qlinearize:
    # Returns the number of common chains of two nodes.
    return po.quicksum(model.disk_in_same_group[disk,peer,group] for group in model.groups)
  else:
    return po.quicksum(calc_disk_in_same_group(model, disk, peer, group) for group in model.groups)

def peer_recovery_traffic_upper_bound(model, disk, peer):
  if self.balanced_incomplete_block_design:
    return calc_peer_recovery_traffic(model, disk, peer) == self.max_recovery_traffic_on_peer
  else:
    # The upper bound of number of common chains of two nodes must <= the max number of targets need to be recovered
    # from a peer node.
    # This constraint guarantees that the chains are distributed as much as possible.
    return calc_peer_recovery_traffic(model, disk, peer) <= self.max_recovery_traffic_on_peer + self.relax_ub
    
model.peer_recovery_traffic_upper_bound_eqn = po.Constraint(model.disk_pairs, rule=peer_recovery_traffic_upper_bound)
```

```python
def peer_recovery_traffic_lower_bound(model, disk, peer):
  # The lower bound of number of common chains of two nodes.
  # Two nodes could share no common chain.
  return calc_peer_recovery_traffic(model, disk, peer) >= max(0, self.max_recovery_traffic_on_peer - self.relax_lb)

model.peer_recovery_traffic_lower_bound_eqn = po.Constraint(model.disk_pairs, rule=peer_recovery_traffic_lower_bound)
```

In summary, these snippets defined the range of the number of common chains between two nodes. They want the chains distributed across the cluster as evenly as possible.

## Object

The script is designed to find a valid solution rather than a best solution in a limited time. So it only defines a dummy object:

```python
model.obj = po.Objective(expr=1)  # dummy objective
```

## Output

If a solution is found, the results are written into two files using python pickle format.

## Conclusion

The script is designed to find solutions for the following things with specific constraints:

1. How many chains need to be created based on user's setup?
2. How to distribute these chains across the cluster as evenly as possible?
