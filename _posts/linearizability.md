What is linearizability?

As if the data is a single copy? Recency.
If a person sees Vi at time T,
then at time T + t, Vi + j (j >= 0)

appears as if there is a single copy of data

After any read has returned the new value, all following reads (on the
same or other clients) must also return the new value.

Linearizability is a recency guarantee on reads and writes of a register

Locks owned by a single node. All nodes must agree which node owns the lock 

How to implement a linearizable system?
Consensus alg. can be used to implement linearizable storage

Leader selection = > relies on a linearizable storage


Total order broadcast?


What happens if you do not provide linearizability?