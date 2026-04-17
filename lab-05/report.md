# Large Scale Computing - CAP
This report briefly explains how the system behaves under partitions for CP (AtomicLong) and AP (PNCounter) structures, showing CAP trade-offs.

## Ex 1
Initially all nodes return 0 and after increment on node 1, all show 1 because Raft replicates the update to all members. After partition, node 3 is isolated and cannot reach quorum (2/3), so both reads and writes fail there (timeouts). Nodes 1 and 2 still form a majority, so they can read and update (value becomes 2). After healing, Raft synchronizes state and node 3 catches up to value 2.

```
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  0
  2     |  0
  3     |  0
[OK]> add:1
  Node 1 += 1 -> 1
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  1
  2     |  1
  3     |  1
[OK]> partition
  Node 3 isolated from Nodes 1 and 2. (Majority: nodes 1,2)
[SPLIT]> get:3
  Node 3 -> (timeout - no quorum)
[SPLIT]> get:1
  Node 1 -> 1
[SPLIT]> get:2
  Node 2 -> 1
[SPLIT]> add:1
  Node 1 += 1 -> 2
[SPLIT]> getAll
  Node  |  Value
  ------+--------
  1     |  2
  2     |  2
  3     |  (timeout - no quorum)
[SPLIT]> add:3
  Node 3 += 1 -> (timeout - no quorum)
[SPLIT]> getAll
  Node  |  Value
  ------+--------
  1     |  2
  2     |  2
  3     |  (timeout - no quorum)
[SPLIT]> heal
  Partition healed. All nodes can communicate.
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  2
  2     |  2
  3     |  2
```

## Ex 2
Before partition everything works normally and values are consistent. After splitting all nodes, no subset has majority (each node is alone), so no quorum exists anywhere. Because of this, all reads and writes fail on every node (timeouts). After healing, the cluster restores quorum and continues with the last committed value (1), since no updates were accepted during the partition.

```
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  0
  2     |  0
  3     |  0
[OK]> add:1
  Node 1 += 1 -> 1
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  1
  2     |  1
  3     |  1
[SPLIT]> get:1
  Node 1 -> (timeout - no quorum)
[SPLIT]> add:1
  Node 1 += 1 -> (timeout - no quorum)
[SPLIT]> get:2
  Node 2 -> (timeout - no quorum)
[SPLIT]> add:2
  Node 2 += 1 -> (timeout - no quorum)
[SPLIT]> get:3
  Node 3 -> (timeout - no quorum)
[SPLIT]> add:3
  Node 3 += 1 -> (timeout - no quorum)
[SPLIT]> heal
  Partition healed. All nodes can communicate.
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  1
  2     |  1
  3     |  1
```

## Ex 3
Initially all nodes converge to value 1 after increment due to replication. After partition, all nodes remain available: both the isolated node and majority side can read and update independently because no quorum is required. Node 1 and node 3 both increment locally (to 2), while node 2 stays at 1. After healing, CRDT merge combines all increments (1 initial + 1 from node 1 + 1 from node 3), so all nodes converge to value 3.

```
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  0
  2     |  0
  3     |  0
[OK]> add:1
  Node 1 += 1 -> 1
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  1
  2     |  1
  3     |  1
[OK]> partition
  Node 3 isolated from Nodes 1 and 2.
[SPLIT]> get:3
  Node 3 -> 1
[SPLIT]> get:1
  Node 1 -> 1
[SPLIT]> get:2
  Node 2 -> 1
[SPLIT]> add:1
  Node 1 += 1 -> 2
[SPLIT]> add:3
  Node 3 += 1 -> 2
[SPLIT]> getAll
  Node  |  Value
  ------+--------
  1     |  2
  2     |  1
  3     |  2
[SPLIT]> heal
  Partition healed. All nodes can communicate.
[OK]> getAll
  Node  |  Value
  ------+--------
  1     |  3
  2     |  3
  3     |  3
```

## CP vs AP
CP data structures require the Raft algorithm because they must ensure strong consistency and linearizability by replicating operations through a consensus mechanism (majority agreement). Raft guarantees a single agreed order of updates and prevents conflicting writes. After partition heal, Raft reconciles logs and ensures all nodes converge to the same consistent state. AP structures do not need Raft because they prioritize availability and resolve conflicts using CRDT merge logic instead of consensus.

## Session guarantees
Read-Your-Writes (RYW) means a client always sees its own updates, even if the system is only eventually consistent. Monotonic reads mean once a client sees a value, it will never see an older value later. These are session guarantees because they apply per client session (not globally), ensuring a consistent experience for a single user despite weak consistency across the system.