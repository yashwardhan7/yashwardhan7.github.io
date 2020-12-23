---
layout: post
title:  "Notes on Distributed Systems Design"
date:   2020-12-22
categories: distributed-systems
excerpt_separator: <!--more-->
---
last updated: 2020-12-22
<!--more-->

## Key characteristics of distributed systems

* **R**eliability: ability to handle component failure.
  * achieved by adding redundancy and eliminating SPOFs (single point of failure).
  * reliability implies high availability over extended period of time, taking into account various modes of failure.
* **E**fficiency
  * measured using rps (requests per second) and latency (95%ile and avg).
* **A**vailability
  * measured as percentage of overall time that system is available.
  * 99.9% availability implies 44mins of downtime per month.
  * 99.99 (4 9s) implies 4.4 mins of downtime per month.
  * an un-reliable system can have high availability if the system can recover quickly, automatically.
* **M**anageability
  * ease of maintenance, diagnosis and repair.
* **S**calability: ability to grow to accommodate increased demand.
  * Horizontal: scale by adding more servers connected over a network.
    * easier to manage dynamically.
  * Vertical: scale by adding more resources to 1 server.
    * scaling requires downtime and comes with an upper limit.

## CAP Theorem

* **C**onsistency: guaranteeing that a read followed by write returns the correct value.
* **A**vailability: negligible downtime.
* **P**arition: ability to withstand network outage or errors.

A system can provide any 2 of these 3 characteristics. For distributed systems partitioning is a must-requirement, so
they can either provide: A-P or C-P. Distributed system design must compromise between availability or consistency. Most
commonly distributed systems fall on a spectrum or allow tuning between providing high-consistency to no-consistency,
with availability adjusting accordingly.

## Load balancing

A load balancer (LB) helps spread incoming requests across a cluster of servers to reduce load on an individual server
and hence improve latency and reliability (by eliminating SPOF). A smart LB can keep track of health and performance of
each downstream server and either route away traffic from offline servers or de-prioritize unhealthy servers (those
experiencing some kind of upstream outage or performance bottleneck). An LB can be an integrated (eg: NetScaler) or
purely-software solution (eg: HAProxy).

An LB can be placed b/w any 2 components. Common places to add an LB:

* b/w client and client facing service (say, web server).
* b/w client facing service and an internal service.
* b/w a service and database.

### Common load balancing methods

* Round robin
* Weighted round robin
  * weight can be decided based on a combination of server latency, active-connection count, processing capacity.
* Client IP hash

An LB can be a SPOF. To overcome this, we can add redundancy to have 1 active and 1 passive LB where passive takes over
in case of active's outage.

## Caching

Caching helps improve performance by taking advantage of locality of reference principle: recently requested data is
likely to be requested again. Caches can exist at various levels but they provide most benefit when placed nearest to
the client so that they can return data quickly without taxing downstream components.

### Different caching strategies

* App server cache: caches responses for requests (eg: Redis, Memcache).
* Database cache: caches result of db queries.
* CDN: caches static data (photo, video, css, js, html). CDNs are reverse proxies. Client's request to site for a static
  asset is routed through a CDN edge server. If the edge server has a cached copy of the asset it returns it directly to
  client without forwarding the request to origin server (site). Otherwise, the edge server fetches the asset from
  origin server (site), caches it and then forwards the response (requested static asset) to client.

### Cache population strategies

* Read-through or Write-around cache: write to db and invalidate cached value, fetch data in cache on first subsequent
  read.
* Write-through cache: write to db and cache at the same time.
* Write-back cache: write to cache, eventually write to db.

There are several possible cache eviction policies like LRU (Least Recenly used), LFU (Least Frequently Used), 
FIFO, LIFO, Random. However, there rarely is a good reason to use anything apart from LRU.

## Data-partitioning

Breaking up a big database into smaller parts to accommodate growth.

### Partitioning schemes

* Vertical partitioning: tables for different features are stored in different dbs. Works well for data that doesn't
  need to joined too often. eg: user table in userDb and timeline-data table in timelineDb.
* Horizontal partitioning (or  data-sharding): rows are stored in different db servers (called shards) depending on the
partitioning key value.
  * Co-resident sharding: Multiple shards on same machine. Used when hitting vertical scaling limit is not a concern and
    we want to improve read/write performance by distributing indexes over various shards. Co-resident shards can be
    changed to remote shards as necessary (if we're approaching vertical scaling limits).
  * Remote sharding: Each shard on a different machine. Helps make full use of vertical scaling limits.

#### Sharding (or horizontal partitioning) techniques

* Hash-based: Keys spread uniformly over a set of shards. eg:

  ``` java
  getShard(key) => hashNumber(key) % numberOfShards
  ```

  ##### can be improved using Consistent hashing
  
  Basic idea: Take a circular ring with H points (H is the hashing factor, typically H=MAX_INT). Place shards at various
  points on this circular ring. A shard's location on the ring is computed as `shardPos = hashNumber(shardName) % H`.
  To figure out which shard must hold a data-row, first compute the `rowPos = hashNumber(rowKey) % H`. The row must
  then be placed on the first shard with shardPos >= rowPos.
  Adding or removing a shard results in **re-balancing** of shardCount/TotalRows rows on average, which is significantly
  better than simple hash-based sharding where almost all rows need to be re-balanced. Also **to alleviate any concern
  about overloading one shard**, we can place virtual replicas of each shard at various locations on the ring to make the
  data-distribution more uniform. Consistent hashing scheme is commonly used to implement a distributed hash table by
  various databases. Even for those dbs that don't provide consistent hashing, we can easily implement it on the client
  side.

  Detailed reading: https://www.toptal.com/big-data/consistent-hashing

* Range-based: each shard holds items with a range of keys. eg:

  ``` java
  getShard(key) => keyRangeToShardMap.find((keyRange, shard) => keyRange.contains(key))
                                     .map ((keyRange, shard)) => shard)
  ```

* Directory-based: a lookup-directory (usually provided as a micro-service) holds the mapping of keys to shards.

  ``` java
  getShard(key) => lookupService.getShard(key)
  ```

#### Sharding limitations

* JOINs: Because data is distributed over different servers, it is not possible to do JOINs efficiently.
  JOINs can still be done by collating data into a central server but at significant perf cost. A workaround to this
  limitation is to de-normalize the data so JOINs are not required anymore but this can lead to data consistency issues.

* Referential integrity: Foreign key constraints cannot be maintained in data split over various shards. Workaround is
  to move referential integrity enforcement in application code or async data-sanitization jobs.

* Re-balancing: We need to re-balance data onto more shards if it grows beyond the vertical scaling limits of individual
  shards. This may sometimes even involve changing the sharding function or the key. This re-distribution
  can lead to performance issues or even an outage while it is happening. Consistent hashing or Directory-based 
  sharding help minimize these issues.

#### How to use consistent hashing to do re-balancing with no downtime

* When a new shard is added or deleted, (shardCount/TotalRows) rows are moved between shards.
* To add a new shard:
  * Determine the position of newShard as `newShardPos = hashNumber(newShardName) % H`. Then iterate through all rows of
    adjacent next shard and copy those rows whose rowPos <= newShardPos to new shard.
  * While this copying is happening, update clients to inform them of the new-shard insertion. Clients will use the
    new-shard to read/write data and retry reads on adjacent next shard.
  * Once copying of all rows to new shard is complete, mark the new shard as added and update clients. Clients will then
    not retry read on adjacent next shard. Also, delete copied rows from adjacent shard.
* To delete an old shard:
  * Copy all rows in shard to be deleted to next adjacent shard.
  * While this copying is happening, update clients to inform them of the shard deletion. Clients will use the adjacent
    shard to write data and retry reads on shard being deleted.
  * Once copying of all rows to adjacent shard is complete, mark the old shard as deleted and update clients. Clients
    will then not retry read on shard being deleted.

#### How to use directory-based sharding to do re-balancing with no downtime

* To add a new shard:
  * In lookup-service: Define a new shard-lookup function, so now we have 2 shard-lookup functions: getShard(key),
    getShard2(key).
  * Inform clients that a new shard is being added. Clients will use the getShard2 to read/write data and retry reads 
    using getShard function.
  * Copy all rows in existing shards to new shards. For example, given a row with key k1 copy it from shard=getShard(k1)
    to shard=getShard2(k1).
  * Once copying of all affected rows is complete:
    * Update lookup-service to swap getShard and getShard2.
    * Inform clients that new shard has been added. They will then only use getShard function to read/write rows and 
      stop retrying reads.
    * Delete rows duplicated in multiple shards. For example, given a data-row with key k1 delete it from
      shard=getShard2(k1), if getShard(k1) != getShard2(k1).

## Distributed consensus

Distributed consensus involves multiple servers agreeing on a value. Typical consensus algorithms consider the decision
final when a majority of servers (quorum, say 3 out of 5) are available and in agreement. Example applications: electing
a leader, deciding to commit a transaction, state machine replication and atomic broadcasts.

Consensus protocols come in 2 flavors: leader-based and leaderless. Leader-based protocols are easier to implement, but
leaderless protocols provide higher availability. Leader-based protocols (eg: Paxos, Raft) are typically used in
strongly consistent distributed databases, while leaderless protocols are used in blockchain distributed ledgers.

### Paxos

Paxos makes leader election an implicit outcome of reaching agreement through data replication. It is hard to understand
and implement. 

### Raft

Paxos has been practically superseded by Raft. Raft is easier to understand and implement and is [more widely used](https://raft.github.io/#implementations).
It treats leader election as a separate pre-requisite step before any data replication can be done. This allows
for dynamic membership changes with node addition and removal. Also, in absence of an unplanned failure or planned
membership change, leader election can be skipped altogether. Note that leader election is automatic in case of a
failure or membership change. Raft also requires that only the nodes with up-to-date data can become leaders.

## Databases

### ACID

* **A**tomicity: guarantee that all operations in a transaction are treated as a unit. Either all are performed or none.
* **C**onsistency: ensures that db is always in correct state. eg: all indexes should get updated correctly after a
  write query. Note that *Consistency* in ACID is different than *Consistency* in CAP.
* **I**solation: guarantee that a transaction does not affect/hinder another.
* **D**urability: guarantee that once completed a transaction is persisted permanently.

### SQL, NoSQL, NewSQL or DistributedSQL

#### How DistributedSQL works

Data is sharded across a multi node cluster. Each node has a SQL processor and distributed query execution engine.
Each node can accept any input request, distribute that request to other nodes and then aggregate the result to return
the response. The db cluster supports strongly consistent data replication and multi-row ACID transactions to provide
an experience as if working with a single SQL database.

#### Comparison

|                         |SQL                      |NoSQL                    |NewSQL or Distributed SQL           |
|-------------------------|-------------------------|-------------------------|------------------------------------|
| **Key characteristics** | ACID compliance         | Horizontal scaling      |ACID + Horizontal scaling           |
| **CAP**                 | CA                      | Mostly AP, some CP      | CP                                 |
| **Performance**         |                         |                         |                                    |
| **Schema**              | Carefully designed      | None                    | Carefully designed                 |
|                         | Change requires downtime|                         | Dynamically changeable             |
| **Query language**      | SQL                     | Custom for each vendor  | SQL                                |
| **Flavors**             |                         | Key-value store         |                                    |
|                         |                         | Document db             |                                    |
|                         |                         | Graph db                |                                    |
|                         |                         | Wide-column table       |                                    |
| **Examples**            | MySQL, Oracle, Postgres | S3, RocksDb (KV)        | MySQL NDB Cluster                  |
|                         |                         | MongoDB,CouchDB (docdb) | Vitess (mysql-based, open-src)     |
|                         |                         | Neo4j, Neptune (graph)  | Citus (postgres-based, open-src)   |
|                         |                         |Cassandra,HBase (widecol)|Yugabyte (postgres-compat, open-src)|
|                         |                         | DynamoDb                | CockraoachDb (postgres-compat)     |
|                         |                         |                         | AWS Aurora (mysql, postgres-compat)|
|                         |                         |                         | Google Spanner (proprietory-SQL)   |
|                         |                         |                         | Apple FoundationDb                 |
|                         |                         |                         | Azure CosmosDb                     |
|                         |                         |                         |                                    |
|                         |                         |                         |                                    |

## Polling, Long-Polling and WebSockets

### Polling

Client polls server over HTTP as needed. Issue: Not very efficient as most of the time server responds back with empty
response.

### Long polling

Client establishes a long lived HTTP connection. Server replies when a response is ready. Then client closes the
connection and immediately establishes a new one. Uses XMLHttpRequest API. 

Issue: 1 way communication, long lived HTTP connection remains passive most of the time and wastes server resources, 
long lived connection may get timed out or disrupted by a proxy.

### Server sent events

Client establishes a long lived HTTP connection. Server replies multiple times over the same connection as necessary.
Performance when using HTTP/2 is comparable with Websockets. Uses EventSource API. 

Issue: 1 way communication, some HTTP overhead.

### Websockets

New protocol. 1 duplex connection. Client and Server can talk as necessary. Uses Websocket API. 

Issue: New protocol so development overhead, Websocket security needs to be handled separately from HTTP security.
Less compliant with existing IT infrastructure like Load Balancer, Firewall, etc.

## How to go about designing a distributed system

Start from the most basic design and expand on it in a breadth first manner, that is, add sub-components and 
complexity while thinking about how to get to next stage (from the point of view of a startup that's acquiring 
customers at fast pace).

* Define the problem statement and list out requirements: functional and operational (performance, reliability).
* Estimate traffic and storage volume, at each step of growth.
* Describe high level design:
  * Define public APIs.
  * Define data model: schema, type of db.
  * Describe various components and how they interact.
* Dive into details:
  * Break down high-level components into sub-components.
  * Describe algorithms to solve sub-problems.
  * List out use-cases that can cause performance or reliability issues.
  * Describe how to handle those and improve performance and reliability in general (eliminate SPOFs):
    * Choosing an appropriate data-partitioning scheme.
      * commonly used technique: sharding with consistent hashing.
    * Caching: CDN or DHT (and at what layers). What and how much. 
      * eg: 20% of data is accessed 80% of time, so caching 20% of overall data can significantly improve performance.
    * Using Load balancing (among micro-services and db-replicas).
    * Monitoring: how and what. 
      * eg: latency, availability, requests-per-second, infra-load, cache-stats.
      * Monitor both the external facing and internal services. 

## Glossary

* **downstream** dependency of a component (say, serviceA): Another component that depends-on or calls serviceA.
* **upstream** dependency of serviceA: Another component that is used-by or called-by serviceA.
* LB: load balancer.
* SPOF: single point of failure.


## More reading

#### Blogs

* [highscalability](http://highscalability.com/all-time-favorites/)
* [educative](https://www.educative.io/path/scalability-system-design)
* [donnemartin](https://github.com/donnemartin/system-design-primer)
* [toptal](https://www.toptal.com/developers/blog)
* [twitter](https://blog.twitter.com/engineering/en_us.html)
* [8bitmen](https://www.8bitmen.com/)
* [Werner Vogels - Amazon CTO](https://www.allthingsdistributed.com/)

#### on Databases

* [How database indexes work](https://www.eandbsoftware.org/how-a-database-index-can-help-performance/)
* [Comparing databases](https://docs.yugabyte.com/latest/comparisons/)
* [Common ACID misconceptions](https://blog.yugabyte.com/6-signs-you-might-be-misunderstanding-acid-transactions-in-distributed-databases/)
* [LSM vs BTree](https://blog.yugabyte.com/a-busy-developers-guide-to-database-storage-engines-the-basics/): LSM gives
  faster write, BTree gives faster random read. LSM more suited for write heavy scenarios (modern cloud) and reads can
  be optimized to give performance comparable to BTree.
* [Why RocksDB](https://blog.yugabyte.com/how-we-built-a-high-performance-document-store-on-rocksdb/)
* [Cassandra usage by examples](https://www.eandbsoftware.org/apache-cassandra-best-practices-and-examples/)
* [ACID with DynamoDB](https://github.com/awslabs/dynamodb-transactions/blob/master/DESIGN.md)
* [Sharding](https://www.acodersjourney.com/database-sharding/)

#### on CDN
* [How does a CDN work](https://www.keycdn.com/support/how-does-a-cdn-work)

#### Interesting whitepapers

* [Dynamo whitepaper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
* [Kafka whitepaper](http://notes.stephenholiday.com/Kafka.pdf)
* [Consistent hashing whitepaper](https://www.akamai.com/es/es/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)
* [Paxos whitepaper](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/paxos-simple-Copy.pdf)
* [Chubby whitepaper](http://static.googleusercontent.com/media/research.google.com/en/us/archive/chubby-osdi06.pdf)
* [Concurrency controls whitepaper](http://sites.fas.harvard.edu/~cs265/papers/kung-1981.pdf)
* [Gossip whitepaper](http://highscalability.com/blog/2011/11/14/using-gossip-protocols-for-failure-detection-monitoring-mess.html)
* [Zookeeper whitepaper](https://www.usenix.org/legacy/event/usenix10/tech/full_papers/Hunt.pdf)
* [MapReduce whitepaper](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf)
* [Hadoop whitepaper](http://storageconference.us/2010/Papers/MSST/Shvachko.pdf)

#### Talks

* [Dropbox](https://www.youtube.com/watch?v=PE4gwstWhmc)
