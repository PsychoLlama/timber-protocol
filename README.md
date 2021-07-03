<div align="center">
  <h1>Timber</h1>
  <p>
    An algorithm for replicating an operation log peer-to-peer, supporting safe checkpointing and compaction.
  <p>
</div>

## Purpose

Synchronizing data between computers is a difficult problem. It's a difficult problem because it's a distributed systems problem. Updates arrive out of order, people modify the same resource concurrently, computers go offline, and edge cases lead to replica divergence and data corruption. Even databases struggle with replication, each flavor with its own disappointing tradeoffs.

There's an active sector of research surrounding *safe* general-purpose data replication. It centers on Conflict-free Replicated Data Types (CRDTs) and Operational Transform (OT). OT relies on central sources of truth while CRDTs are much more flexible, even useful in peer-to-peer settings. The downside is space complexity: CRDTs grow in size correlating with the richness of the data types.

Although some specialized algorithms have better performance, CRDTs usually cost expressiveness, increased size, and inability to delete data.

Timber is an attempt at a general-purpose CRDT built on an operation log supporting *true delete*. The operation log allows custom events, providing the full freedom of an event-sourcing architecture, while the novel checkpointing algorithm enables safe, distributed compaction, effectively removing all CRDT metadata past the lower bound.

The only caveat is limited offline support. If multiple peers are editing concurrently while another is offline, the offline peer may need to download and replay their events after regaining connection.

## Technical Overview

### Operation Log

Timber uses a causal tree to hold the operation log. Causal trees model ordering of concurrent and sequential events. They look something like this:

```
Two concurrent editors, names "A" and "B". Both start at lamport time=1.

                                       +-------+
                                  +----> B: t5 |
                                  |    +-------+
                 +-------+  +-----+-+
            +----> B: t2 +--> A: t4 |
            |    +-------+  +-----+-+
      +-----+-+                   |    +-------+  +-------+
Root  | A: t1 |                   +----> A: t5 +--> B: t6 |
      +-----+-+                        +-------+  +-------+
            |    +-------+  +-------+
            +----> A: t2 +--> A: t3 |
                 +-------+  +-------+
```

The diagram shows a causal tree containing several operations. Each operation holds a reference to the last one, kind of like a linked list. When you add an operation you always append to the most "recent" item.

Branches happen because it takes time for updates to propagate over the network. If two people append simultaneously, obviously that creates a branch, as shown with `A@2` and `B@2`.

Once each actor learns about the other and see the divergence, which branch should they consider the most "recent"? Well, it doesn't matter, it only matters that they agree. Choose a deterministic function. In the diagram `B@2` was considered more recent, so `A@4` continued there.

Now, because branches are ordered and each transaction references the last, we can derive a linear history:

```
A@1, A@2, A@3, B@2, A@4, B@5, A@5, B@6
```

This creates the operation log.

Merging with other peers is as simple as a union between two sets of operations.

### Checkpointing and Compaction

Checkpointing is the other piece, but it's more complex. We need a way to snapshot state at a point in time and prune out the old operations, otherwise the log grows unbounded.

Imagine a simple log that naively edits a string: it has 386 `Insert` and 67 `Delete` operations with new ones arriving every second. If you don't prune the log you'll eventually run out of space, but if you throw away the old ones, how will you know if you've seen them before? Plus, string operations are order-dependant, and you just deleted the ordering.

There is no way to recover. Compaction just ruined the replica.

The problem is about establishing a lower bound. At some point, we expect the older updates to have propagated through the system. Nobody's building on that first operation anymore. So the question becomes, how do we all agree on a shared point in causal history? What point in the log has everyone seen?

Answering that question is key. That shared point in history is the checkpoint, and any operations before it can be replaced with the state they produce. Simple enough.

Unfortunately, answering that is a problem of distributed consensus. That's what makes it complicated.

The obvious choice is a [vector clock](https://en.wikipedia.org/wiki/Vector_clock). If each operation includes a checksum of the tree and the last known operation, we can use the clock to establish the lower bound. Further, this information is generated deterministically from the causal tree. Given the same tree, everyone sees the same clock.

```
// Lower: 5, Upper: 7

Map {
  actor_a => [operation_6, '<tree-checksum>'],
  actor_b => [operation_5, '<tree-checksum>'],
  actor_c => [operation_7, '<tree-checksum>'],
  actor_d => [operation_6, '<tree-checksum>'],
  actor_e => [operation_7, '<tree-checksum>'],
}
```

Sounds good in theory, but we're still missing an edge case. Establishing that global lower bound assumes our clock captures *everyone*. There could be a contributor out there who drops the lower bound to `operation_3` who's updates just haven't reached us. Running compaction up to operation #5 will break our replica.

We need a way to create a vector clock that's representative of every peer in the system. This brings us to the core of Timber: The Committee.

### The Committee

Writes are restricted to a set of peers called "the committee". When you create a new dataset you are the only one in the committee, the only one allowed to write, and each operation is signed by the author's private key.

```
committee = Map {
  dataset_creator => pub_key,
}
```

New collaborators are added by invitation only. Invites are processed by appending an `Invite` operation to the causal tree, like so:

```
      +-------------------------+    +-------------------------+
Root  | invite_member(pub_key1) +----> invite_member(pub_key2) |
      +-------------------------+    +-------------------------+
```

This would add two more members to the committee.

```
committee = Map {
  dataset_creator => pub_key,
  invited_member1 => pub_key1,
  invited_member2 => pub_key2,
}
```

But those invites don't take effect yet. They're marked pending and the new collaborators still can't write. Invites are only valid after they pass the checkpoint, which if you recall from the last section, requires every peer to acknowledge the operation.

The first invite happens almost instantaneously because the set of peers only contains one author, the creator. A checkpoint applies instantly and the new editor (`pub_key1`) is added to the committee. The next invite requires acknowledgement from both peers. Waiting for acknowledgement ensures everyone has time to update their vector clocks, which prevents the premature compaction problem.

There are circumstances where waiting for an invitation to process isn't worth the time. You may consider allowing committee members to act in a client/server relationship, accepting writes on behalf of others. But that is out of scope.

This solves the problem of safe compaction, but leaves open a hole: what happens if someone permanently drops offline? Without another mechanism to account for it, the compaction process would halt and no new peers could be invited. This brings us to the next step, peer eviction.

### Peer Eviction

For the sake of understanding the eviction cycle, assume everyone in the committee contributes an equal amount (the mechanism will be explained later). Operations are divided into "rounds", defined by the number of contributors. Say there are 15 contributors: one round is 15 operations, two rounds is 30, three is 45... you get the idea.

If everyone acknowledges a history where a contributor is noticeably silent for a predetermined number of rounds, they're considered inactive and the eviction phase begins.

It would be tempting to immediately evict the member and pass a checkpoint, but remember, not everyone crosses the checkpoint simultaneously. It would open a race condition where contributors could reach two different states:

1. The silent contributor posts an update which drastically raises the lower bound, triggering a normal checkpoint.
2. The last contributor acknowledges the silence, permanently kicking the silent member and invalidating potential updates from scenario #1.

If anyone integrated a change from scenario #1 before the silence was acknowledged, they would permanently diverge from the rest of the cluster. Naturally we can't let that happen, so once again we return to the dance of consensus.

The eviction phase is a period of suspended compaction. The silent contributor is placed on probation and requires an endorsement before their permissions are reinstated. Everyone still accepts their updates, but without an endorsement, they have no effect: they aren't counted toward a checkpoint, they aren't considered when deriving state. It's as though they weren't added at all.

There are two ways the eviction phase can end:

1. Everyone acknowledges the silence and the inactive contributor is finally removed. Checkpoints continue normally.
2. At least one person received an update from the silent contributor and endorsed it, nullifying the eviction. The peer is reinstated.

Each member only gets one vote: acknowledge silence or submit an endorsement. Either way, the eviction will end after everyone submits their next update.

Endorsements are just an extra bit of metadata attached to your typical updates. They point to another operation ID proving that the peer in question has actually been active. Any operation that doesn't include an endorsement is considered an acknowledgement of silence. If you receive an update that exonerates a supposedly inactive peer but you've already declared silence, you aren't allowed to write an endorsement. The best you can do is quickly forward it to someone else who hasn't voted yet.

If the eviction is nullified by a valid endorsement, the peer is reinstated. They can submit operations again and any pending updates that were ignored by those who acknowledged silence become validated, which is to say they are counted when deriving state and computing checkpoints. On the other hand, if the peer is evicted, those pending updates can safely be purged.

Endorsements and eviction phases work around the race condition by explicitly representing both cases in the protocol. The silent contributor still has a chance to redeem itself and the network still has the power to drop inactive participants.

Now, all that covers the perspective of the peers who remain online, but what about those offline still attempting to edit? From their perspective *everyone else* went silent. They might try to evict the rest of the network. Well, that's where an exception kicks in. You can only evict a peer if at least 51% of the committee acknowledged the silence.

That exception means that unless your partition controls the majority, you cannot evict peers, and therefore you cannot run compaction. This is a good thing. It means once you regain network connection, you still have the entire list of operations. You can integrate them upstream or rebase them after the compaction point. No data gets lost.

## Distribution of Activity

Remember the assumption from the last section?

> assume everyone in the committee contributes an equal amount (the mechanism will be explained later)

Here's why: Transactions are processed by the committee member who's ID most closely matches the operation. But that's a loaded statement. How do you determine the closest ID? Well, each operation is identified by a hash of its contents, and if you take the XOR value between the operation ID and a member ID, you get a numeric distance. It's symmetric and deterministic, and it can be checked by other participants.

This ensures an equal distribution of work that scales dynamically with the committee and reveals patterns in inactivity, which is the basis for peer eviction.

Now unless you've got a controlled node cluster with high availability, it's unrealistic to assume everyone in the committee is always online and able to process transactions. If the member with the closest ID is offline, your operation would not be processable. That's where you can tune the system. Decide on a predetermined redundancy value, and instead of sending the operation to just one member, send it up to `n` closest members.

Check this from the receiver side. If you find `n` members that would've been a better match, either reject it or forward the update to the correct members. If the sender is also in the committee then they should know better.

If you get a duplicate operation signed by another member, keep the one with the more similar member ID.

## Sync Protocol

Many parts of Timber assume local ordering of transactions during synchronization, which is to say that during sync we always get your old updates before your new ones. We should never receive a transaction without a parent. This guarantee is provided by the sync protocol. It interactively exchanges missing transactions, starting from the oldest, until both members share a causal history.

There are two message types:

- `Sync`: Either starts or continues a synchronization.
- `SyncFinished`: Indicates both parties are synchronized.

Synchronization happens any time you send an operation to another peer. It starts by including any new operations along with a list of hashes...

```json
[
  "Sync",
  {
    "interval": 16,
    "hashes": [
      "<hash-1>",
      "<hash-2>",
      "<hash-3>",
      "<hash-4>",
      "<hash-5>"
    ],
    "operations": [
      "<new-operation>"
    ]
  }
]
```

The `interval` field defines how often a hash is captured on a linearized tree. It is a power of 2 to maximize the chance of intersection with other intervals, which is important if you maintain a cache of checksums.

If there are 29 operations in the causal tree with an `interval=8`, you will have 4 hashes. Adjust the interval at runtime to optimize for size vs latency.

On the receiving side, after integrating the transactions, the client confirms the incremental checksums. If all checksums match, the sync is resolved through a `SyncFinished` message.

If not all checksums match, the client finds the first deviating hash, captures all transactions from the entire interval, and responds with a counter-sync.

The cycle repeats until all checksums match.

> Note how this algorithm would behave if someone added new transactions while synchronization was already in progress.

## Summary

And that's the gist of it. Using the Timber protocol, you should be able to replicate event logs with safe checkpointing and state compaction.

The name is derived from the idea of a forest of (causal) trees, implementing event *logs*, all being cut down (compacted).

> Timber  
> used interjectionally to warn of a falling tree 

If you found this document useful or interesting, give it a star.
