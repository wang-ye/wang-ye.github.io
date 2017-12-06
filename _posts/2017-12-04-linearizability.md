---
layout: post
title:  "Consensus Notes"
date:   2017-12-04 19:59:03 -0800
---

This post is a note about some recent learnings on fault-tolerant consensus  in a distributed system. We will start with some important concepts including consensus, total order broadcast and linearizability, and discuss some theoretical findings on them. Finally, we will briefly talk about Zookeeper, the widely-used service for reaching consensus. The detailed proof and precise definitions can be found in the book [designing big data applications](https://dataintensive.net/). 

## What is Linearizability?
Linearizability first appears as a strong consistency model in 1990s. It says there is a *single global state* that different processes talk to. If a write operation changes a linearizable register from value *old* to *new*, then all *future* read operations on the register will get *new* or some later values. 
Note that, this single global state does not have to be on a single machine - it can also spread on multiple nodes while logically appearing as a *single global state*. 

## What is Total Order Broadcast?
Total Order Broadcast means reliably broadcasting the total orders of operations to all nodes. Each machine will see the same order of operations, although they may receive the broadcast data at different times.

## What is Consensus? 
Consensus means *getting several nodes to agree on something*. This is a very important building block in designing distributed systems. One common use case for consensus is distributed lock: when one node acquires the lock, all other nodes must agree on which node currently owns the lock, and wait before the lock is released. Paxos, Raft and Zab are all well-known consensus algorithms.

## A Profound Finding
Researchers found that consensus is equivalent to total order broadcast, which is also equivalent to a linearizable compare-and-set register. This helps understand what consensus really means in the real world.

## What is Zookeeper?
Zookeeper is a service providing fault-tolerant consensus. With Zookeeper, you can do many things including distributed locks, leader selection and service discovery. Again, to understand more, please refer to the [book](https://dataintensive.net/).
