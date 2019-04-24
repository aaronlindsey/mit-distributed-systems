# In Search of an Understandable Consensus Algorithm (Extended Version)

By Diego Ongaro and John Ousterhout

[Link to Paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

This paper describes Raft, a novel consensus algorithm. Paxos, the dominant consensus algorithm at the time of writing, is notoriously difficult to understand. Furthermore, Paxos lacks quality reference implementations and descriptions to make it understandable for use in a practical system. The authors aimed to improve Paxos by creating an algorithm that was easier to understand and implement while still performing as efficiently as Paxos.

Raft takes a replicated state machine approach to consensus. In this approach, each node in a cluster applies the exact same state transformations to its state machine to maintain a consistent state. Typically, replicated state machines provide the following properties:
  - Safety, i.e. never returning an incorrect result under non-byzantine conditions,
  - Availability, i.e. the cluster remains operational as long as a majority of nodes are operational,
  - Do not depend on timing to ensure consistency, and
  - In the common case, a command completes as soon as a majority of nodes has responded.

*See Figure 2 in the paper for a condensed summary of the Raft algorithm.*

Raft makes the following guarantees:
1.  Election Safety: at most one leader can be elected in a given term.
2.  Leader Append-Only: a leader only appends log entries; never overwrites or appends.
3.  Log Matching: if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index.
4.  Leader Completeness: if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms.
5.  State Machine Safety: if a server has applies a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

A Raft cluster consists of several nodes which can be in one of three states: leader, candidate, and follower. Under normal operation, there is exactly one leader and the rest of the nodes are followers. *See Figure 4 in the paper for a description of the states and their transitions.* Raft divides time into terms of arbitrary length. At the beginning of each term, an election takes place to decide the leader. It is possible for the election to result in a split vote, in which case the term ends immediately and a new term begins. Terms are numbered by monotonically increasing integers which act as a logical clock.

### Leader Election

All nodes begin as followers. If a node does not receive any communication from a leader for a specified timeout, called the election timeout, it will begin an election. Leaders maintain their authority by sending a periodic heartbeat message to all followers.

When a node begins an election, it transitions to candidate state and requests votes from other members of the cluster. It wins if it receives votes from a majority of nodes in the cluster. A node will cast one vote per term for candidates on a first-come first-served basis. If a candidate wins, it transitions to leader state and starts sending heartbeat messages to the other nodes. If a candidate receives legitimate commuication from a leader of a higher-numbered term, it reverts to follower state. Finally, if there are multiple candidates and the votes are split such that no candidate has a majority of votes, the term will end and a new election will start.

Raft uses randomized election timeouts to ensure progress. The timeouts ensure that the vast majority of the time, a single candidate will initiate and win an election before another candidate has a chance to request votes.

### Log Replication

Leaders accept client commands and attempt to replicate them to the followers. After a command is successfully replicated to a majority of followers, the leader considers the command "committed", applies it to its state machine, and responds to the client. Committed commands will eventually be executed on each of the followers. If a follower crashes or is unresponsive, the leader retries replicating the command indefinitely. **How to prevent the system from running out of resources while retrying failed requests?**

Inconsistencies may arise when a leader crashes. Raft handles inconsistencies by forcing the followers' logs to duplicate the leader's log. (This is safe for reasons discussed in the next section.) 

Raft's log replication mechanisms allow it to accept, replicate, and apply new log entries as long as a majority of the servers are up. If a single node becomes slow, it will not impact the overall performance because Raft only needs a majority of nodes to respond to the command.

### Safety

Raft's strategy for ensuring that all nodes execute the same state machine commands entails a restriction on which nodes may be elected as leader:

> Raft uses the voting process to prevent a candidate from winning an election unless its log contains all committed entries. A candidate must contact a majority of the cluster in order to be elected, which means that every committed entry must be present in at least one of those servers. If the candidate’s log is at least as up-to-date as any other log in that majority (where “up-to-date” is defined precisely below), then it will hold all the committed entries. The RequestVote RPC implements this restriction: the RPC includes information about the candidate’s log, and the voter denies its vote if its own log is more up-to-date than that of the candidate.
> Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the logs. If the logs have last entries with different terms, then the log with the later term is more up-to-date. If the logs end with the same term, then whichever log is longer is more up-to-date.

### Cluster Membership Changes

Changes to the cluster configuration are allowed without making the system unavailable. Raft implements this using joint consensus. The basic idea is that for a temporary period of time: commands are replicated to both the old and new clusters, a command must be replicated to separate majorities from both the clusters to be committed, and any server from either configuration may serve as the leader.

### Log Compaction

Raft uses periodic snapshotting to keep the log a maintainable size. Each node does its own snapshotting. During snapshotting, the current state is stored as well as the last included index and term numbers. A special RPC is implemented to allow cloning the snapshotted state from one node to another.

### Implementation and Evaluation

The authors evaulated Raft in terms of understandability, correctness, and performance. They hosted an experiement where participants attended a lectures about Paxos and Raft and were given a quiz afterward. The participants generally scored higher on the Raft quiz. The authors made reasonable attempts to mitigate bias.

The authors provided a formal proof of Raft's safety property.

Raft's performance is similar to other consensus algorithms. It performs replication in the normal case using the minimal number of messages. By tuning timeouts, the authors were able to achieve good performance of leader election.

