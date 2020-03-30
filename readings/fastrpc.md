# Datacenter RPCs can be General and Fast

Anuj Kalia, Michael Kaminsky*, and David G. Andersen (Carnegie Mellon University, *Intel Labs)

_16th {USENIX} Symposium on Networked Systems Design and Implementation_, 2019. (NSDI '19)

[Find original paper using Google Scholar](https://scholar.google.com/scholar?q=Datacenter+RPCs+can+be+General+and+Fast)

One way to achieve high performance when desigining distributed systems is to use specialized networking hardware, like RDMA, lossless networks, FPGAs, and programmable switches. The problem with these approaches is that this specialized hardware is not usually readily available.

This paper presents the design of a general-purpose RPC system, eRPC, that achieves performance comparable to that of systems with specialized hardware. It demonstrates that such specialization is not necessary.

## Implementation

One way eRPC achieves good performance is by ensuring RPC handlers are short. It allows the use of both the dispatch thread and worker threads to run RPC handlers:

> Striking a balance, eRPC allows running request handlers in both dispatch threads and worker threads: When registering a request handler, the programmer specifies whether the handler should run in a dispatch thread. This is the only additional user input required in eRPC. In typical use cases, handlers that require up to a few hundred nanoseconds use dispatch threads, and longer handlers use worker threads.
