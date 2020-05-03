---
title: "Consistency Models under the Looking Glass"
date: 2020-01-31
summary: ""
draft: true
---


## Data-Centric Consistency

Different consistency models ordered from strong to weakest:

 1. **Strict Consistency**:  
    Events ordered by _physical time_.
    Every process sees all reads and writes in the same order.
    Impossible to achieve for distributed systems due to inaccurate clocks.
 2. **Linearizability**:  
    Events ordered by (non-unique) global _logical time_.
    Like Sequential Consistency, uses Lamport timestamps for ordering.
    However, physical time with a bounded error is taken into account as well. Thus, a process will always read the most recent write (except for parallel writes, where we can't be sure about ordering).
 3. **Sequential Consistency**:  
    Read and write events are ordered by Lamport timestamps.
    Beware of concurrent writes, which can not be ordered by that!
    Also, a process might read a very old value from a phyical time perspective.
 4. **Causal Consistency**:  
    A "potential causal relation" between reads and writes is established.
    Concurrent writes may be seen in different order by different processes.
 5. **FIFO Consistency**:  
    Writes from one process are seen by all other processes in the order they were issued.
    However, writes from multiple processes might be interleaved; even differently interleaved in different processes.
 6. **Weak Consistency / Group Consistency**:  
    Requires a dedicated _synchronization point_.
    Before synchronization, reads may appear in any random order.
    After synchronization, all processes will read the same data.
    Processes might need to agree on a value on conflicts during synchronization.


## Client-Centric Consistency

 - **Read Your Writes**:  
   A write operation is always completed by all replicas before a successive read operation by the same process, no matter where the read operation takes place. E.g. a write operation by a client needs to be retrieved by all replicas and only after that client can read this key again.  
   A typical example is updating a password.
 - **Monotonic Reads**  
   If a process reads a value formed by a set of operations `S`, any successive read operation of that process will always return a value formed from a _superset_ of `S`. Thus, a process always sees _more recent data_ (but not necessarily fresh!).
   If a process reads again from another replica that replica must have already received the relevant older operations.
 - **Monotonic Writes**
   The writes by the same process are performed in the same order at every replica. This is similar to FIFO consistency.
 - **Writes Follow Reads**
   Any successive write operation by a process will be performed on a state that is _up to date_ with the value _most recently read_ by that process.