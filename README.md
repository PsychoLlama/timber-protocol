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

The diagram shows a causal tree containing several operations. Each operation holds a reference to the last one, kind of like a linked list. We have branches because it takes time for updates to propagate through the system, and if two users append simultaneously, it creates a branch. The branches are re-joined as soon as each author learns of the other's changes.

Branched operations are ordered. It doesn't matter what order we choose so long as it's consistent (Timber compares the hash IDs). In the case of the diagram, `B@2` was considered greater than `A@2`. Once `A` learned of the updates, it appended to the new branch instead of its own. A similar join happened at `B@5`.

Now, because branches are ordered and each transaction references the last, we can derive a linear history:

```
A@1, A@2, A@3, B@2, A@4, B@5, A@5, B@6
```

This is the operation log.

## Terminology

<dl>
  <dt>Transaction</dt>
  <dd>An event in the operation log.</dd>

  <dt>Block</dt>
  <dd>A signed transaction with causal and actor metadata.</dd>
</dd>

## Sync Protocol

There are two message types:

- `Sync`: Either starts or continues a synchronization.
- `SyncFinished`: Indicates both parties are synchronized.

Synchronization happens any time you send a block to another peer. It starts by including any new blocks along with a list of hashes...

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
    "blocks": [
      "<new-block>"
    ]
  }
]
```

The `interval` field defines how often a hash is captured on a linearized tree. It is a power of 2 to maximize the chance of intersection with other intervals, which is important if you maintain a cache of checksums.

If there are 29 operations in the causal tree with an `interval=8`, you will have 4 hashes. Adjust the interval at runtime to optimize for size vs latency.

On the receiving side, after integrating the blocks, the client confirms the incremental checksums. If all checksums match, the sync is resolved through a `SyncFinished` message.

If not all checksums match, the client finds the first deviating hash, captures all blocks from the entire interval, and responds with a counter-sync.

The cycle repeats until all checksums match.

> Note how this algorithm would behave if someone added new blocks while synchronization was already in progress.

## Related Reading

Archagon's [Data Laced with History](http://archagon.net/blog/2018/03/24/data-laced-with-history/) is an absolutely genius take on CRDTs. It's a long read, but I highly recommend it.

---

A synchronized causal tree with federated checkpointing and safe
compaction.

## Overview
Causal trees can be stored in a grow-only set and replicated through
a union function. When linearized, the tree can be used to represent an
event log. The implications are obvious. The true problem is compaction.

If you compact while concurrent updates were made in a foreign
partition, the replicas irreparably diverge (no common ancestor).
The only way to safely run compaction is to know the lower causal bound
of every actor. That boundary is your compaction point.

Vector clocks are the perfect choice, but if an actor goes offline
permanently (which is highly likely in a federated system) compaction
will never run again. We need a way to safely drop peers from the vector
clock.

## Implementation
Either initialize or clone a dataset. The dataset is a causal tree in
a grow set which derives to a list of peers. The peers are used to
generate a vector clock, structured like this:

```
peer.id => causal_pointer
```

Each peer's causal pointer is only updated when they add a block, signed
by their private key.

The lower bound of the vector clock is your checkpoint. Everything prior
is safe for compaction, and it offers a guarantee that every peer
accepted the blocks before it.

### Evicting Peers
If a peer becomes inactive, they are removed. This is the magic part.

For reasons later explained, it is reasonable to assume each peer
contributes at an equal rate. Therefore, peers are considered "inactive"
if their contribution history is disproportionately silent.

Now, just because we haven't seen updates from that peer doesn't mean
they haven't contributed. We could be experiencing a partition. But that
doesn't matter. What matters is whether our partition is bigger. To
reach certainty, we use the paxos approach and measure by majority.

To determine if the ruling majority considers a peer inactive, order the
peers from the last checkpoint by their causal pointer, newest to
oldest, and group the controlling majority into a new set, computing
a new lower bound. If the distance from the majority to the common bound
is greater than the maximum allowed, the oldest peer is evicted.

Since you've just changed the definition of the majority, repeat until
no peers are evicted.

### Adding Peers
Only peers who've passed a checkpoint can write new transactions.
Actors outside the dataset can ask a member for a referral, and if the
member approves, they append a transaction which contains:

- A combined hash of every preceding pending transaction
- The requester's public key
- The member's ID (hash of their public key)
- A cryptographic signature verifying authenticity
- The preceding transaction ID
- The new transaction ID (hash of the public key + member ID)

They then broadcast this transaction to all their immediate connections,
who also forward the message. Their request is honored after it passes
the checkpoint.

If a receiving member doesn't agree with the combined hash, they ignore
the update and prompt a resynchronization, identifying the missing
blocks since the last checkpoint. This prevents your vector clock from
allowing gaps in causal history which could lead to wrongful eviction.

### Requesting Referrals
In order to get a referral, you must generate a key pair, derive your
member ID, convert it to a number and find `N` dataset members with an
ID nearest yours, then send them your request.

All members of the dataset check this request, and if a more suitable
referrer existed, your request will be rejected. This distribution has
multiple advantages:

- Request load is automatically spread across the network as it grows.
- Comparing by hash provides a uniform distribution, which is why
  members are assumed to contribute in roughly equal proportion.

Since the referral request is sent to multiple members, duplicate
transactions might appear, but they can be ignored. For determinism, the
causally earlier request should always be preferred.

### Storing Data
Arbitrary data can be written, updated, and deleted using the same
causal tree and checkpointing implementation. Simply add support for
custom transaction types and a compaction hook (called on checkpoint).

Data is written using the same referral mechanic, by generating
a transaction ID and sending it to `N` members with similar IDs. Using
the same mechanism doubles as a heartbeat and helps to quickly fill
history gaps across the network.

There's a rare possibility that a checkpoint or eviction just changed
the closest delegates. You may need to retry.

In order to maintain high availability under partition, you're allowed
to instantly save the transaction locally. You must leave the signature
field blank. All peers should exclude it from contribution history and
checkpoints, and it should be retried occasionally. Signed and valid
transactions take priority, deleting the original.

### Minority Partitions
If members fall into a minority partition, they cannot create
a checkpoint because the majority is missing. They cannot elect new
members because election depends on the previous cohort. However, they
can continue to add new transactions, they'll just never reach
a checkpoint.

This is useful behavior. It means they don't fork off permanently, and
no data is destroyed under compaction. When the partition is lifted,
they can re-request to join the majority and rebase their pending
transactions.

Transactions that require strong consistency can wait for the
checkpoint.

### Network Connections
Each member connects with `N` other members in a Kademlia fashion.
Immediate connections are used for propagating updates.

Peers outside the checkpoint are ignored.

### Usage Scenarios
In theory, this scales from a few personal devices to thousands of
independent actors. The eviction criteria implicitly scales availability
requirements based on the number of members.

At the low end, syncing data between your laptop, phone, and home server
can work well. Devices can be offline for months at a time without being
evicted, and even if they are, resynchronization can be handled
automatically.

At the high end with thousands of actors, the slightest amount of
downtime might miss enough opportunities to get you evicted, losing your
write access. Members must pull their own weight to keep their status.
