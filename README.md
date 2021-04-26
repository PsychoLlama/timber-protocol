# Compacting Causal Tree
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