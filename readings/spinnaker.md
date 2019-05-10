# Using Paxos to Build a Scalable, Consistent, and Highly Available Datastore

By Jun Rao, Eugene J. Shekita, and Sandeep Tata

[Link to Paper](https://pdos.csail.mit.edu/6.824/papers/spinnaker.pdf)

Automatic sharding and load balancing have emerged as a cost-effective and manageable way to scale large distributed data stores. However, with sharding comes a need for replication to ensure fault tolerance. Master-slave replication is not sufficient for tolerating failures due to the fact that the master must reject client requests if its slave is unavailable to ensure consistency. In addition, with only two nodes (as in master-slave replication) there is a risk of double-disk failure leading to data loss.

One alternative to master-slave replication is 3-way replication. In this model, additional care is needed to maintain consistency between the replicas. Some data stores opt for an "eventual consistency" model which forces the application to deal with consistency issues that might arise. This has the advantage of being extremely available, but most applications would rather have stronger consistency guarantees and support for transactions than extreme availability.

Spinnaker is a distributed key-value store featuring 3-way replication with either strong or *timeline* consistency. The use of timeline consistency for reads allows potentially stale data to be returned in exchange for better performance. It uses a Paxos-based protocol for replication.

The paper's main contributions are:
- Describing how Paxos can be used in a real system,
- Demonstrating that Paxos-based replication can be simpler and more performant than previously assumed
- Showing that Spinnaker is as fast or faster on reads and only 10-15% slower on writes than an eventual consistency data store, and
- Showing that Spinnaker's design leads to a highly-available system

*Q: The authors argue that Spinnaker is a CA system when in fact it does tolerate network partitions to some degree. Wouldn't it instead be a CP system?*

A Spinnaker cluster is comprised of nodes that store data sharded using key-based range partitioning. A group of nodes responsible for replicating a key range is called a *cohort*. Each node has a disk-backed log, an in-memory commit queue, in-memory data structures to store committed data, and SSTables to persist committed data to disk. LSNs are used to uniquely identify and order log entries.

## Replication

Replication happens on a per-cohort basis. Under normal conditions each cohort has one leader. Figure 4 shows a sequence diagram of a client write, summarized below:
1. Leader receives client write W
1. Leader appends log entry for W to disk; in parallel, leader appends W to its commit queue and sends a propose message for W to its followers
1. Follower receives propose message for W
1. Follower appends log entry for W to disk, appends W to commit queue
1. Follower replies with ack to leader
1. After leader receives ack from majority of followers, it applies W to its memtable which commits W
1. Leader returns response to client

A leader periodically sends an async commit message to its followers asking them to apply writes up to a specific LSN to their memtable, committing the writes. Writes always go through the leader. Reads, however, may go to either the leader or one of the followers. If strong consistency is desired, reads must go through the leader. If timeline consistency is preferred, clients may read from the followers and receive writes that have been committed to the memtable.

## Recovery

Recovery is implemented in two phases, local recovery and catch-up. Local recovery is the process of each cohort member re-applying committed entries from its own log to its memtable. Catch-up is the process of each follower sending its last committed LSN to the leader and the leader sending all committed writes after the LSN to that follower.

When a new leader is chosen, it uses the algorithm in Figure 6 to catch followers up to its latest LSN.

## Leader Election

Leader election is triggered when an existing leader fails or when the entire system is starting up. Figure 7 explains the algorithm used for leader election. In summary, each cohort member proposes its last LSN to the other members. After a majority of proposals have been received, the member with the highest LSN is chosen to be the new leader. No committed writes will be lost because a committed write has to be forced to the logs of a majority of members, and a majority of members have to participate in the leader election. Therefore, there will be at least one member which as the latest committed write that also participates in the election.

Spinnaker relies on Zookeeper to store data related to the leader election in a reliable and safe way.

## Discussion

A cohort is available for writes and strongly consistent reads as long as a majority of cohort members are available. It is available for timeline consistent reads if only one of the members is available. There is a possibility for lost writes if a majority of members permanently fail in rapid succession.

Spinnaker trades some degree of performance, scaling, and availability for strongly consistent writes and reads. These trade-offs are examined by the experiments.

## Experiments

Spinnaker consistent reads and timeline reads are compared to Cassandra's weak and quorum reads, respectively. Cassandra's weak read accesses one node and its quorum read accesses two nodes to check for inconsistencies. Spinnaker's consistent reads have significantly lower latency and are able to handle more load than Cassandra's quorum reads. Cassandra's weak read has slightly lower latency than Spinnaker's timeline read while they handle about the same amount of load.

Spinnaker writes are compared with Cassandra's quorum writes, which provide the same replication degree. The latency of Spinnaker's writes is 5% to 10% worse than Cassandra due to the fact that a write in Spinnaker has to go to the cohort leader and one follower while in Cassandra it can go to any two nodes.

Overall, Spinnaker's design choice of using a single leader does have a measurable negative impact on its performance and scalability compared to Cassandra. However, many applications might be willing to make this trade-off for the better consistency offered by Spinnaker.

## Conclusion

Spinnaker uses Paxos-based replication to ensure consistency and high-availability. It demonstrates that Paxos-based replication can be kept relatively simple and achieve acceptable performance.
