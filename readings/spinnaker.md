# Using Paxos to Build a Scalable, Consistent, and Highly Available Datastore

By Jun Rao, Eugene J. Shekita, and Sandeep Tata

[Link to Paper](https://pdos.csail.mit.edu/6.824/papers/spinnaker.pdf)

Automatic sharding and load balancing have emerged as a cost-effective and managable way to scale large distributed datastores. However, with sharding comes a need for replication to ensure fault tolerance. Master-slave replication is not sufficient for tolerating failures due to the fact that the master must reject client requests if its slave is unavailable to ensure consistency. In addition, with only two nodes (as in master-slave replication) there is a risk of double-disk failure leading to data loss.

One alternative to master-slave replication is 3-way replication. In this model, additional care is needed to maintain consistency between the replicas. Some datastores opt for an "eventual consistency" model which forces the applicaiton to deal with consistency issues that might arise. This has the advantage of being extremely available, but most applications would rather have stronger consistency guarantees and support for transactions than extreme availability.

*Q: The authors argue that Spinnaker is a CA system when in fact it does tolerate network partitions to some degree. How is this not a CP system?*