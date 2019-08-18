# No compromises: distributed transactions with consistency, availability, and performance

By Aleksandar Dragojević, Dushyanth Narayanan, Edmund B. Nightingale, Matthew Renzelmann, Alex Shamis, Anirudh Badam, Miguel Castro

[Link to Paper](https://dl.acm.org/ft_gateway.cfm?id=2815425&type=pdf)

Distributed transactions with strong consistency and high availability greatly simplify building distributed systems. However, they usually come with a price — namely performance. The authors introduce FaRM, which achieves 140 million TATP transactions per second on 90 machines with a 4.9 TB database and recovers from failure in less than 50 ms. The key to the authors' success is exploiting two new hardware trends to reduce storage and network bottlenecks: non-volatile DRAM and fast commodity networks with RDMA.

## Hardware Trends

Non-volatile DRAM can be achieved by attaching distributed uninterruptable power supply (UPS) units to traditional volatile DRAM.

> "When a power failure occurs, the distributed UPS saves the contents of memory to a commodity SSD using the energy from the battery. This not only improves the common-case performance by avoiding synchronous writes to SSD, it also preserves the lifetime of the SSD by writing to it only when failures occur."

The cost of adding distributed UPS units to traditional DRAM is lower than the cost of non-volatile DIMMs.

> "Therefore, it is feasible and cost-effective to treat all machine memory as non-volatile RAM (NVRAM). FaRM stores all data in memory, and considers it durable when it has been written to NVRAM on multiple replicas."

FaRM's use of NVRAM helps reduce storage and network bottlenecks. This tends to expose CPU bottlenecks, which the authors address by using one-sided RDMA operations.

> "One-sided RDMA uses no remote CPU and it avoids most local CPI overhead." 

## Architecture

Clients interact with FaRM using an API that allows access to objects within transactions. FaRM takes an optimistic concurrency control approach to transactions, whereby conflicts are checked at commit-time instead of at each operation. Committed transactions are strictly serializable. Lock-free reads are also supported by the FaRM API. Configuration data is stored in a Zookeeper cluster.

## Failure Recovery

FaRM uses replication to ensure durability and high-availability.

> "We provide durability for all committed transactions even if the entire cluster fails or loses power: all committed state can be recovered from regions and logs stored in non-volatile DRAM. We ensure durability even if at most [all but one] replicas per object lose the contents of non-volatile DRAM."

> "FaRM can also maintain availability with failures and network partitions provided a partition exists that contains a majority of the machines which remain connected to each other and to a majority of replicas in the Zookeeper service, and the partition contains at least one replica of each object."

## Evaluation

Two benchmarks were used for performance evaluation:
- TATP: A read-dominated transactional benchmark
- TPC-C: A database benchmark with more complex transactions accessing hundreds of rows.

### Normal Case Performance

> "FaRM performs 140 million TATP transactions per second with 58 µs median latency and 645 µs 99th percentile latency."

> "FaRM performs up to 4.5 million TPC-C "new order" transactions per second with median latency of 808 µs and 99th percentile latency of 1.9 ms.

These results are vastly better than any other published performance results using these benchmarks.

### Failure Performance

For TATP transactions:

> "We measured recovery time from the point where the failed machine is suspected by the CM until throughput recovers to 80% of the average throughput before the failure. The median recovery time is around 50 ms and in more than 70% of the executions the recovery time is less than 100 ms. In the remaining cases, the recovery took more than 100 ms, but always less than 200 ms."

