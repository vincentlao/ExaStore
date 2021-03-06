#Single Node Storage Engine Design

Our altimate goal is to provide a low cost distributed and replicated storage
system. This document explains the design of the most important component:
the single node storage engine.

## Key Value API

The Storage Engine is a stand alone exe accepting requests via UDP. We provide
a library `ExaBroker` that knows how to communicate with the storage engine.

### Data Record

The data record should have the following fields:

- Key: 128 bits.  Custom keys can be hashed into 128b integer with minimum collision. 
- Owner: a 64 bit hash key.  1..232-1 reserved. Only an owner can overwrite or delete.
- Reader: a 64 bit hash key.  0 means anyone.  1..232-1 reserved.
- Time Stamp: the time when the written request arrives primary server. The store supports versioning based on time stamps.
- Parts: boolean. Any non-zero value indicating the data is one part of a series of write partial requests.
- Values: data blobs, currently limited to 16 MB in 1MB parts.

### POLICIES

Policies regarding data retention and consistency are maintained based on 64 bit Owner ID.

**Expiration lifetime:**  unsigned integer limited to 2047 hours.  Zero means Zero
(will not be written to SSD). In Exabyte scale we cannot allow data to live forever.
All data expires in a finite time.  In future we may make provisions for extending
lifetime, for discard prior to schedule, for copying to archive, etc.  These functions
will revolve around the capabilities associated with the Owner ID.

**Allow Secondary Read:** Boolean. When true, on the rare occasion of Primary server
timeout, the read operation is sent to a secondary server.

### API

Write.

> Atomic operation: Write a blob, under a key, with specified owner and permitted reader, size.
  
Read.

> Always reads the whole value.  
> To be implemented. Reader can provide a time stamp in order to retrieve an earlier version.

 
Delete.

> Atomic operation. Allow the key-value to expire.  Do not retrieve the value if key is presented and the
request has no time stamp earlier than the deletion.

WritePartial/WriteFinal.

> Atomic operation, for opportunistic collection of a burst of small related data with multiple parallel
writers, e.g. in distributed tracing. Results will be atomic but multiple blobs concatenated with
undefined order.  Timestamp and expiration lifetime will relate to the first part accepted.
Multiple Finals are idempotent.  If none is seen, the Final is implied by a timeout.

> Opportunistic in the sense that we provide no guarantee. In most cases all partial writes within
5 seconds of the first part accepted will be collected and merged. After merging, all subsequent
partial write to the same key will be ignored . Under unusual circumstances they might drop at 1
second. We aim at achieving 5 seconds 90% of the time. Time window of the session (from the first
accepted part to the merging time) will be reported to read requests.

> Example scenario for this case is session based logging, where all writes (originating from
multiple servers) would send events keyed on a session ID and they concatenate.  Allowing for 
logging to include pathological slow cases this would probably work find with a 5 second window 
(even if we discard beyond 5 seconds, we probably captured what the pathology was).


## Infrastructure

Exabyte Store strives at high-level of control and low resources consumption. There are some
key design decision in the infrastructure of the system which enables this.

### UDP Networking

In order to avoid holding hundreds even thousands of TCP connection open, we choose to use UDP
as communication protocol. The plan is to use 64kB per-segment limit, since that should be small
enough to slip through the switches without buffer overflows even when a collision occurs.

### Activities And Continuations

One design principle we adopted is a sequential execution model, where an activity runs
sequentially like in a single threaded model. 

This came from the Turns model in Midori.
The Midori tradeoff was that a single activity might not go as fast as if you could make
it truly hog many threads.  But in practice they went really fast because nothing needed
locks, and not much ordinary code is really good for parallel except IO and special data
patterns, neither of which needed the ordinary code to be parallel. In return, scaling was
amazing.  You could load a ton of stuff onto a Midori server and it still made progress.

To simulate a similar model, we implements sequential Activities, where each Activity
is a sequence of interleaving Continuations and I/O operations (disk or RPC). And each
Continuation is a sequence of non-blocking operations. All Continuations in an Activity
execute sequentially without overlapping, like in a single thread. Context switch does
not happen unless waiting for async I/O.

This sequential model will remove the needs for locking within an Activity. An example
Activity is processing of write request:

- (Continuation: add request to update queue) 
- (I/O: send replication requests to other machines and wait for responses) 
- (Continuation: update store)
- (Continuation: update Catalog)
- (I/O: send ack to client)

### IOCP, Scheduling
 
We use IOCP to implement a job scheduler (think of a thread pool). See class SchedulerIOCP. 

We can achieve good efficiency with IOCP, combined with the sequential Activity model
described above. A thread should run a Continuation (i.e. a non-blocking synchronous
action), and when finished it goes back to get another Continuation off the IOCP queue.
This looks at first glance like other forms of multi-threading but the thread never
changes context, is never forced to be swapped.  It just pulls items off a queue,
an operation which even including locking is far cheaper than a context swap.  Of
course context swaps do still occur since IOCP will pause the thread if the queue
is empty (but, in that case we are idle so the cost of the pause is not painful).
In the IOCP model we can clearly control how many threads we run.  

This can vary a bit according to how many micro-partitions we support on a server,
although probably the variation is not too wide since it should be set to handle
bursts of traffic and, as a background service, generally the bursts will be spaced 
apart and queues of requests should be rare.  We just need to be sure we have enough
threads to drain down any queue faster than the arrival of more requests.

