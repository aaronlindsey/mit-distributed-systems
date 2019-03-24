# The Design of a Practical System for Fault-Tolerant Virtual Machines

By Daniel J. Scales, Mike Nelson, and Ganesh Venkitachalam

[Link to Paper](http://nil.csail.mit.edu/6.824/2018/papers/vm-ft.pdf)

VMware vSphere Fault Tolerance (FT) is a system for providing fault-tolerant virtual machines. A typical setup has a primary VM, a backup VM, and a shared virtual disk. VMware FT models the VMs as deterministic state machines that are kept in sync by starting them from the same initial state and ensuring that they receive the same input requests in the same order. Non-deterministic operations (such as interrupts and reading the clock cycle counter) are replicated deterministically by recording all necessary information about the operation. The current implmentation does not support multi-processor VMs because of performance issues that arise because almost every access to shared memory is non-deterministic.

Replication between the primary VM and backup VM is accomplished using a log channel that sends input received by the primary, information about non-deterministic operations, and output operations to the backup. The log messages are normally sent asynchronously, although there is a rule that says the primary VM may not send any output to the external world until the backup has received and acknowledged the log entry associated with the output operation. This rule is to ensure consistency in the output produced by the VMs. There is nothing to prevent an output from being sent more than once, but the TCP protocol already handles duplicate packets so this is fine.

*It's unclear to me if all of the log messages have to be acknowledged by the backup or only the output operations. I think maybe the acknowledgement of the output operation implicity acknowledges the operations leading up to the output operation.*

Failover happens when either of the following happen for longer than a specific timeout:
1.  The UDP heartbeat check between the primary and backup (or vice versa) fails, or
2.  Logging traffic has stopped

The split-brain problem is avoided by having the VM that wants to go live (quit replication and become an independent VM) perform a test-and-set operation on the shared virtual disk. If the operation succeeds, the VM can go live. If it fails that means the other VM has already gone live and the VM will halt itself.

The authors implemented a number of features and optimizations to make this system ready for a produciton workload:
  - VMware FT will automatically start a new backup VM if one of the VMs has gone live.
  - It will automatically slow down the execution of the primary VM if the primary and backup get too far out of sync.

It is also possible to use VMware FT in a non-shared disk mode which is useful with the VMs are far apart. This makes the split-brain problem more complicated to deal with though and the authors don't detail a solution but propose using a third party server that both servers can talk to.

Experiments show that VMware FT adds less than 10% performance degradation for a variety of workloads. Workloads with a lot of inbound IO will use large logging bandwidth because the received data has to be transferred to the backup. Outbound IO uses less bandwidth because the sent data does not need to be transferred. One way to reduce logging bandwidth for high inbound IO workloads is to have the backup VM perform its own disk reads instead of sending them over the log channel. (Only works in the non-shared disk mode.)    

