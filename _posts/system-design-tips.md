# Read/Write API split
Optimize read operations?
On the other hand, Flickr shards the users based on geo locations. This does not conduct read/write split.


# How to optimize for hot data?
Using cache, but how to determine data is hot? From query logs - from offline or domain knowledge.

# Storage
Why no-sql supports high IOPS?