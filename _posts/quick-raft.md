What is Raft?

Consensus algorithm

Why Raft?

What is consensus? 
Replicated state machines.
A cluster of machines

Fault tolerance

How to understand it?

There is a missing weapon. The person who get the weapon will be the king of the world. Once you become the king, you make orders periodically and everyone has to follow! BTW, everyone wants to become the king.

If after sometime, a person does not receive the order in a given amount of time, 

Why leader? 
Term - leader selection + commit actions
Leader election:
safety: at most one leader exists at any time.
timeout -> candidate