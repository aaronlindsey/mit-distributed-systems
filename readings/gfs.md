# The Google File System

By Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung

[Link to Paper](https://ai.google/research/pubs/pub51)

Google File System (GFS) is a distributed file system used at Google. It's widely used within Google and by other companies via the open source Hadoop File System (HDFS) which is based on GFS.

The designers of GFS make the following assumptions:
  - The file system will be run on a cluster where nodes often fail,
  - The files stored on the system will typically be large (MB-GB),
  - Most files are mutated by appending new data rather than overwriting existing data, and
  - Most files are read sequentially all at once.

There is a single active master and many chunkservers. The master stores metadata for the file system in memory (it's state is also persisted to disk). The chunkservers store the actual data. Files are divided into fixed-size chunks stored on the chunkservers. Chunks are replicated between the chunkservers to a degree specified by the application.

The flow of a write is as follows:
1.  The client asks the master which chunkserver holds the lease for a particular chunk and the locations of the replicas. The master grants one chunkserver the lease unless one already has it.
2.  The master replies and the client caches the reply until the chunkserver no longer holds the lease or becomes unreachable.
3.  The client pushes data to the replicas linearly with pipelining. For example, the client starts pushing data to replica. As soon as the replica starts receiving data it starts forwarding it to the primary. Repeat to all the replicas.
4.  After the replicas acknowledge receiving the data, the client sends a write request to the pimary. The primary serializes the mutations (possibly received by multiple clients) and applies them to its state.
5.  The primary forwards the write to replicas. They apply mutations in the same order as the primary.
6.  Once the replicas are done, they reply to the primary which replies back to the client. The primary includes any errors that would have left the data in an inconsistent state and the client may try again.

There is also an atomic append operation which is similar to the above write, except the client does not specify the offset. The primary will atomically append the data and instruct the replicas to append the data at the same offset. If failure, the data is left in an inconsistent state and the client tries again.

The consistency model enforces strong consistency for file namespace mutations because there is a single master. The consistency model for data mutations depends on the type of mutation, whether it succeeded or failed, and whether mutltiple clients are mutating the same chunk:

|                      | Write                    | Record Append                          |
| -------------------- | ------------------------ | -------------------------------------- |
| Serial success       | defined                  | defined interspersed with inconsistent |
| Concurrent successes | consistent but undefined | defined interspersed with inconsistent |
| Failure              | inconsistent             | inconsistent                           |

Applications deal with inconsistency and duplicates:
  - For record append, use UUID to detect duplicates and checksums to detect invalid data
  - For regular write, create temporary files and then atomically renaming them
  - For regular write, create checkpoints within files to mark defined regions

In experiments, this design produces high aggregate throughput for large sequential reads and writes.

#### Summary (copied from MIT 6.824 notes):

Case study of performance, fault-tolerance, consistency
  - Specialized for MapReduce applications

What works well in GFS?
  - Huge sequential reads and writes
  - Appends
  - Huge throughput (3 copies, striping)
  - Fault tolerance of data (3 copies)

What works less well in GFS?
  - Fault-tolerance of master
  - Small files (master a bottleneck)
  - Clients may see stale data
  - Appends maybe duplicated
