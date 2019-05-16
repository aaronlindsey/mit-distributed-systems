# ZooKeeper: Wait-free coordination for Internet-scale systems

By Patrick Hunt, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed

[Link to Paper](https://www.usenix.org/event/usenix10/tech/full_papers/Hunt.pdf)

ZooKeeper is a "coordination service" which exposes an API that allows application developers to implement their own coordination primitives using wait-free data objects. (Wait-free in this context seems to mean non-blocking.)

The API allows manipulating data objects called *znodes* which have the following properties:
- They are organized hierarchically (like a file system)
- They can be either *regular*, meaning they must be created and deleted explicitly, or *ephemeral*, meaning ZooKeeper will remove them automatically when the client session that created them terminates or times out.
- Clients can use *watches* to subscribe to notifications of changes to a znode without using polling

Examples of wait-free operations provided by the API:
- **create(path, data, flags)**: creates a znode; flags specifies the type regular or ephemeral
- **delete(path, version)**: deletes the znode if it is at a specific verison
- **exists(path, watch), getData(path, watch), setData(path, watch), getChildren(path, watch)**: perform operations on znodes and return a watch
- **sync(path)**: waits for all updates pending at the start of the operation to propagate to the server the client is connected to

ZooKeeper guarantees that writes are linearizable (serializable and respect precedence) and that a client's operations are executed in FIFO order. Without these guarantees, it would not be very useful for implementing coordination primitives that applications can rely on.

It turns out, these simple primitives are sufficient for implementing a wide variety of common distributed systems coordination primitives:
- **Configuration management**: Store config in znode and all processes obtain watch to be notified of config changes
- **Rendezvous**: Client creates znode and passes it to system as startup param; processes fill in config details
- **Group membership**: Processes each create one ephemeral znode under a group znode; when a process fails, its znode is removed; processes can call getChildren to obtain list of members and may watch this to be notified of membership changes

Interestingly, even though ZooKeeper itself only provides wait-free operations, it can be used to implement lock-based concurrency primitives. These are described in the paper:
- Simple locks
- Simple locks without herd effect
- Read/write locks
- Double barrier

Some examples of applications that use ZooKeeper:
- **Yahoo! Fetching Service**: web crawler that uses ZK for configuration metadata and leader election
- **Katta**: distributed indexer that uses ZK for group membership, leader election, and config management
- **Yahoo! Message Broker**: pub/sub system that uses ZK for config metadata, failure detection, and group membership

Interesting details of ZooKeeper's implementation and performance characteristics:
- Reads do not have to go to the cluster leader and therefore may return stale values. This has the benefit of high performance (throughput) for reads but it has the drawback of relaxed consistency which might be harder for applications to deal with. ZK provides the sync command to allow clients to wait until the read server it's connected to is up-to-date.
