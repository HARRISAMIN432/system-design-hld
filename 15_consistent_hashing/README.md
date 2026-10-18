## Table of Contents

1. [The Problem: A Single Database Becomes a Bottleneck](#1-the-problem-a-single-database-becomes-a-bottleneck)
2. [Horizontal Scaling and Data Partitioning](#2-horizontal-scaling-and-data-partitioning)
3. [What Is a Hash Function?](#3-what-is-a-hash-function)
4. [The Modulo-Based Partitioning Approach](#4-the-modulo-based-partitioning-approach)
5. [The Problem with Adding or Removing Servers](#5-the-problem-with-adding-or-removing-servers)
6. [What Is Consistent Hashing?](#6-what-is-consistent-hashing)
7. [How a Hash Ring Works](#7-how-a-hash-ring-works)
8. [Adding a Server to the Ring](#8-adding-a-server-to-the-ring)
9. [Removing a Server from the Ring](#9-removing-a-server-from-the-ring)
10. [Virtual Nodes](#10-virtual-nodes)
11. [Finding the Keys Affected by a Topology Change](#11-finding-the-keys-affected-by-a-topology-change)
12. [Benefits and Trade-offs](#12-benefits-and-trade-offs)
13. [Hotspots and Load Distribution](#13-hotspots-and-load-distribution)
14. [Where Consistent Hashing Is Used](#14-where-consistent-hashing-is-used)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. The Problem: A Single Database Becomes a Bottleneck

Imagine an application with a large number of users and a growing amount of data. Initially, the application may store all of its data in a single database.

As traffic and data volume increase, that database can become a bottleneck:

- It must handle a growing number of reads and writes.
- Its CPU, memory, storage, or network capacity may become insufficient.
- A single instance limits the system's ability to increase capacity by adding machines.

One way to address this is **horizontal scaling**: add more database servers and distribute the data and workload across them.

For example, instead of one database handling all the workload, deploy three database partitions:

- Database 0
- Database 1
- Database 2

The system must now answer an important question:

**For a given user or data key, which database should store the data, and how will the application find it later?**

The assignment rule must be deterministic. If data is written to one partition but a later lookup checks a different partition, the application may incorrectly behave as though the data does not exist.

## 2. Horizontal Scaling and Data Partitioning

**Horizontal scaling** means increasing capacity by adding more machines or instances rather than only upgrading a single machine.

When a dataset is distributed across multiple databases, the process is commonly called **partitioning** or **sharding**. Each partition stores a subset of the overall data.

For example:

| User ID | Example user | Assigned partition |
|---:|---|---:|
| 1 | Piyush | Database 1 |
| 2 | John | Database 2 |
| 3 | Jane | Database 0 |
| 4 | Alex | Database 1 |
| 5 | Tiger | Database 2 |

The particular assignments depend on the partitioning strategy. What matters is that the system can reproduce the same assignment when it needs to retrieve a user's data.

A partitioning strategy should ideally:
- Be deterministic, so the same key maps to the same destination under an unchanged server configuration.
- Distribute keys and workload reasonably well.
- Make it practical to add capacity or handle server failures.
- Let the system identify the destination for a key without searching every partition.

## 3. What Is a Hash Function?

A **hash function** takes an input—such as a user ID, cache key, or record key—and maps it to a hash value.

In a partitioning system, that value can be used to determine where data belongs.

A simple routing function might look conceptually like this:

```text
partition = hash(user_id) mapped to the available partitions
```

A useful routing hash should be **deterministic**: the same input produces the same result when the hash function and its configuration remain unchanged.

For example, if user ID `1` maps to Database 1 during a write, a later lookup for user ID `1` should calculate Database 1 again, provided the partitioning configuration has not changed.

### Why a random assignment does not work

Suppose the application selects a random partition every time it stores or retrieves data:

1. A write for user ID `1` randomly selects Database 2.
2. A later read for user ID `1` randomly selects Database 0.
3. Database 0 does not contain the record, so the lookup fails.

The issue is not that the record was necessarily lost. The routing decision was inconsistent.

### Determinism and distribution

A routing hash has two separate responsibilities:

1. **Consistent routing:** the same key should lead to the same destination under the same configuration.
2. **Reasonable distribution:** keys should be spread across the available destinations so that one partition does not receive a disproportionate share of the workload.

A function that always returns the same partition is deterministic but distributes data poorly. Determinism alone does not guarantee balanced load.

> **Terminology note:** A routing hash is not necessarily a cryptographic hash. Cryptographic properties such as preimage resistance are usually not required for ordinary partition selection. The important concerns here are stable mapping and distribution.

## 4. The Modulo-Based Partitioning Approach

A straightforward strategy is to compute the remainder after dividing a key's numeric identifier by the number of available servers.

For numeric IDs, the rule is:

```text
partition = user_id % number_of_servers
```

Here, `%` represents the modulo operation: the remainder after division.

Suppose there are three partitions, numbered `0`, `1`, and `2`.

| User ID | Calculation | Partition |
|---:|---|---:|
| 1 | `1 % 3` | 1 |
| 2 | `2 % 3` | 2 |
| 3 | `3 % 3` | 0 |
| 4 | `4 % 3` | 1 |
| 5 | `5 % 3` | 2 |

The same ID maps to the same partition as long as the number of partitions remains three.

For instance, user ID `4` maps to partition `1` because `4 % 3 = 1`. When the application looks up that user later, it can repeat the calculation and find the same partition.

### What this approach gets right

- It is simple to implement.
- It is deterministic for a fixed server count.
- It can distribute numeric keys reasonably well when the key values and hash function have suitable distribution.

### The central limitation

The number of servers is part of the mapping formula. If the number of servers changes, the result may change for many existing keys.

That creates the **re-sharding problem**.

## 5. The Problem with Adding or Removing Servers

### 5.1 Adding a server

Suppose the system initially has three partitions and uses:

```text
partition = user_id % 3
```

Later, the system grows and a fourth partition is added. The formula becomes:

```text
partition = user_id % 4
```

Consider what happens to the existing keys:

| User ID | With 3 servers | With 4 servers | Mapping changed? |
|---:|---:|---:|---|
| 1 | 1 | 1 | No |
| 2 | 2 | 2 | No |
| 3 | 0 | 3 | Yes |
| 4 | 1 | 0 | Yes |
| 5 | 2 | 1 | Yes |

Some keys still map to the same partition by coincidence, while others map elsewhere. In a large dataset, many records may need to be relocated or otherwise made discoverable at their new destinations.

The application cannot simply change the formula and assume the existing records have moved. The routing rule and the physical location of stored data would no longer agree.

### 5.2 Removing a server

The same problem occurs in reverse. If a server is removed and the system changes the modulo divisor, keys may map to different partitions. The application must account for the records that were stored on the removed server and for other keys whose mapping changes.

If a server fails unexpectedly, its data may also need to be recovered or replicated, depending on the system's durability design. Consistent hashing helps determine new ownership; it does not, by itself, guarantee that data has been replicated or recovered.

### 5.3 Why this becomes expensive

In a system with millions of keys, a topology change can cause a large amount of data movement and associated work:

- Reading existing records from their current locations.
- Writing records to new locations.
- Updating routing or ownership metadata where applicable.
- Consuming network bandwidth and storage I/O.
- Managing the extra load generated while rebalancing.

The modulo strategy can be useful in simple or fixed-size deployments, but its mapping changes significantly when the server count changes.

**The goal of consistent hashing is to minimize how many keys change owners when the set of servers changes.**

## 6. What Is Consistent Hashing?

**Consistent hashing** is a partitioning technique that maps both servers and data keys into a shared hash space, commonly represented as a ring.

Instead of calculating a partition by taking a key modulo the current number of servers, the system:

1. Hashes each server's identifier to determine its position on the ring.
2. Hashes each data key to determine its position on the same ring.
3. Starts at the key's position and moves clockwise until it encounters the first server.
4. Assigns the key to that server.

The important difference is that server count is not simply substituted into a modulo expression for every key. When a server is added or removed, only keys in the affected part of the ring generally need to change owners.

This makes consistent hashing particularly useful for systems that need to scale or adapt to changing server membership.

## 7. How a Hash Ring Works

### 7.1 Create a circular hash space

Imagine a clock. Positions are arranged around a circle, and moving clockwise eventually returns to the starting point.

A consistent-hashing ring works similarly. The actual hash space may be very large, but a small illustrative ring can use positions from `1` to `12`.

The circular shape is a convenient way to represent an ordered hash space whose end wraps around to its beginning.

### 7.2 Place the servers on the ring

Each server has an identifier, such as a server ID, hostname, or IP address. Hash that identifier and place the server at the resulting position.

For example, assume the hash positions are:

- Server A at position `2`
- Server B at position `5`
- Server C at position `9`

These values are illustrative, not the output of a specific production hash function.

### 7.3 Hash each data key

When a data key arrives, hash it using the same hash space. Suppose a key hashes to position `3`.

Starting at position `3`, move clockwise until reaching the first server. The first server encountered is Server B at position `5`, so that key is assigned to Server B.

If another key hashes to position `10`, the search continues clockwise, wraps around the end of the ring, and reaches Server A at position `2`. That key is assigned to Server A.

### 7.4 Example assignments

Using servers at positions `2`, `5`, and `9`:

| Key's hash position | First server clockwise | Assigned server |
|---:|---:|---|
| 1 | 2 | A |
| 3 | 5 | B |
| 6 | 9 | C |
| 10 | 2 (after wrap-around) | A |
| 11 | 2 (after wrap-around) | A |

The key point is that a key belongs to the first server at or clockwise after its hash position. Exact equality is assigned to that server if a key and server share a position, subject to the implementation's collision-handling rules.

### 7.5 Reads and writes use the same rule

To store a key, calculate its hash position and find its first clockwise server. To retrieve it, repeat the same calculation using the same ring membership and routing rules.

The system therefore does not need to remember a random destination for every request. It derives the destination from the key and the ring configuration.

In a distributed system, all participating clients or routing components must use a consistent view of ring membership. If different components use different ring configurations, they may route the same key to different servers.

## 8. Adding a Server to the Ring

Suppose the ring has servers at positions `2`, `5`, and `9`, and a new server is added at position `12`.

Before the new server existed, keys hashing to positions after Server C at `9` and before the wrap-around to Server A at `2` belonged to Server A. This includes positions such as `10` and `11`.

After Server D is added at position `12`, keys in the interval after Server C (`9`) through Server D (`12`) now encounter Server D first. Those keys can move from Server A to Server D.

The important observation is that keys in other intervals still encounter the same first server as before. They generally do not need to move.

### Before and after

**Before adding Server D**

- Positions after `2` through `5` belong to Server B.
- Positions after `5` through `9` belong to Server C.
- Positions after `9` through the wrap-around to `2` belong to Server A.

**After adding Server D at `12`**

- The interval after `9` through `12` belongs to Server D.
- The remaining intervals keep their existing owners.

Thus, only the keys in the interval whose ownership changed need to be redistributed. The server is inserted into the ring, and ownership is recalculated for the affected range.

This is the central advantage of consistent hashing: **adding a server typically moves only a fraction of the keys instead of remapping the entire dataset.**

## 9. Removing a Server from the Ring

Removing a server is the reverse operation.

Suppose Server D at position `12` is removed. Keys that were assigned to Server D now move clockwise to the next available server, which in this example is Server A at position `2`.

Keys assigned to Server B and Server C remain assigned to those servers, because their first clockwise server has not changed.

This reduces unnecessary movement, but the system still needs to handle the removed server's data correctly. A practical deployment must consider replication, durability, failure recovery, and how ownership changes are propagated to clients.

Consistent hashing determines the new logical owner of a key; it does not automatically transfer bytes, repair replicas, or guarantee that a failed server's only copy of data remains available.

## 10. Virtual Nodes

A basic consistent-hashing ring can still distribute load unevenly.

### 10.1 The uneven-distribution problem

Hashing server identifiers to positions is not guaranteed to place servers evenly around the ring. Two servers may land close together, leaving one server responsible for a much larger interval than another.

Because keys are assigned to the first server clockwise from their hash positions, a server responsible for a large interval may receive substantially more keys and traffic.

For example, if Server A owns a large portion of the ring while Server B owns only a small portion, Server A may become overloaded even though the system has multiple servers.

### 10.2 What is a virtual node?

A **virtual node** (or vnode) is a logical position on the ring that points to a real physical server.

Instead of representing each physical server at only one point, the system represents it at multiple positions. For example:

- Physical Server A may own virtual nodes A1, A2, and A3.
- Physical Server B may own virtual nodes B1, B2, and B3.
- Physical Server C may own virtual nodes C1, C2, and C3.

All virtual nodes belonging to a physical server route to that same physical machine. They are not separate servers and do not require separate physical deployments.

### 10.3 Why virtual nodes help

Virtual nodes provide several benefits:

- **More balanced ownership:** multiple positions give each physical server several smaller ranges rather than one large range.
- **Reduced sensitivity to a single hash position:** the load depends on several positions rather than only one.
- **More flexible rebalancing:** ranges owned by a server can be redistributed across other servers when membership changes.
- **Support for heterogeneous machines:** implementations can assign different numbers of virtual nodes to servers with different capacities, when configured appropriately.

Virtual nodes improve the probability of a balanced distribution; they do not guarantee perfectly equal traffic. Real workloads can be skewed even when key ownership is balanced.

## 11. Finding the Keys Affected by a Topology Change

Consistent hashing makes the affected range relatively straightforward to identify.

### When a server is added

Let the new server be at position `S_new`. Moving counterclockwise from the new server, find the previous server on the ring, `S_prev`.

The keys in the interval **after `S_prev` and up to `S_new`**, moving clockwise, are the keys whose owner changes to the new server. These keys are moved from the server that previously owned that interval.

For example, if `S_prev` is at position `9` and the new server is at position `12`, the affected interval is `(9, 12]`. Keys in that range are reassigned to the new server.

### When a server is removed

Keys owned by the removed server are reassigned to the next server clockwise on the ring. Other ranges ordinarily retain their owners.

### Why this matters operationally

The ring makes the set of logically affected keys local to a particular range. A system can identify those keys rather than blindly remapping every key in the dataset.

The exact mechanism used to transfer data depends on the database or distributed system. A production implementation also needs to coordinate ownership changes and handle reads and writes that occur during migration.

## 12. Benefits and Trade-offs

### Benefits

1. **Less data movement during scaling**  
   Adding or removing a server generally changes ownership only for keys in affected ring intervals.

2. **Simpler incremental scaling**  
   New servers can take ownership of part of the key space without requiring a complete remapping.

3. **Useful for dynamic server membership**  
   The technique fits systems where nodes may be added, removed, or replaced over time.

4. **Compatible with virtual-node-based balancing**  
   Multiple ring positions per physical server can improve distribution and make rebalancing more flexible.

5. **Deterministic routing**  
   Given the same key, hash configuration, and ring membership, the system can reproduce the same owner.

### Trade-offs and limitations

1. **Load is not automatically perfectly balanced**  
   Ring positions may be uneven, and some keys may be much more popular than others.

2. **Virtual nodes add complexity**  
   The implementation must track multiple logical positions and map each one to a physical server.

3. **Membership changes must be coordinated**  
   Different clients using different ring states can disagree about where a key belongs.

4. **Data migration still occurs**  
   Consistent hashing minimizes redistribution; it does not eliminate it.

5. **Replication and availability are separate concerns**  
   Consistent hashing alone does not provide backups, replication, fault tolerance, or data durability.

6. **Hot keys can remain a problem**  
   A single highly requested key may overload its owner even if the keys are evenly distributed across the ring.

## 13. Hotspots and Load Distribution

A **hotspot** occurs when one server or partition receives a disproportionately large share of traffic or work.

There are at least two distinct reasons this can happen:

- **Uneven key ownership:** one server owns a much larger portion of the hash ring than others. Virtual nodes can help mitigate this.
- **Skewed access patterns:** a small number of keys receive a very large number of requests. Even a well-balanced ring may assign a popular key to one server, creating a hotspot.

For example, if many users repeatedly request the same celebrity's profile, the partition holding that profile may receive far more traffic than other partitions. Consistent hashing does not automatically split one hot key across many servers, because the same key normally maps to one owner.

Addressing hot keys may require additional techniques, such as caching, replication for read-heavy data, request coalescing, or application-specific partitioning. The appropriate approach depends on the workload and consistency requirements.

## 14. Where Consistent Hashing Is Used

Consistent hashing is used in distributed systems where keys must be mapped to nodes and node membership can change. Common examples include:

- **Distributed caches:** mapping cache keys to cache nodes while limiting cache reassignment when nodes change.
- **Distributed databases and storage systems:** assigning key ranges or partitions to nodes in a cluster.
- **Distributed routing and request assignment:** selecting a node for a key-based workload.
- **Systems with changing membership:** reducing the amount of reassignment required as machines are added or removed.

Examples of systems and technologies commonly associated with consistent hashing or related partitioning strategies include Amazon Dynamo-style distributed storage designs and Apache Cassandra. The exact partitioning and replication details vary by system and implementation.

## 15. Key Takeaways

- A single database can become a bottleneck as an application grows, making horizontal scaling and partitioning useful.
- A deterministic hash-based routing function helps a system send the same key to the same partition under an unchanged configuration.
- The simple modulo rule `key % number_of_servers` is easy to understand, but changing the server count can remap many existing keys.
- Consistent hashing places both servers and keys in a shared circular hash space.
- A key is assigned to the first server encountered when moving clockwise from the key's position.
- Adding a server generally moves only the keys in the interval newly owned by that server; removing a server transfers its keys to the next server clockwise.
- Virtual nodes give each physical server multiple positions on the ring and can improve load distribution.
- Consistent hashing minimizes redistribution, but it does not eliminate data migration or solve every load-balancing, replication, availability, or hot-key problem.

**In one sentence:** Consistent hashing is a way to distribute keys across a changing set of servers while minimizing how many keys need to move when the system's membership changes.
'''