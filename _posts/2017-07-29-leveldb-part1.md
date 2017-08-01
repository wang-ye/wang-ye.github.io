---
layout: post
title:  "A LevelDB Overview- Part One"
date:   2017-07-28 20:49:03 -0800
---

[LevelDB](https://github.com/google/leveldb) is a well-known open source library for key-value storage, written by two Google fellows Sanjay Ghemawat and Jeff Dean. It unveils some of Google's architecture design concepts such as Bigtable. By study this repository, we can understand more about building a key-value store and writing quality C++ code. Given the complexity of the project, we will divide it into two series. In this blog I will talk about the architecture and some overviews, the second one will focus more on the source code.

## Physical Layout
Let's start with the architecture diagram from [a blog](http://wiesen.github.io/post/LevelDB-Storage/). We can easily see all components of LevelDB - Memtable, Immutable Memtable in memory, and Current, Manifest, log and SSTable on disk. Also, you can get a sense of how the Read, Write and Compaction interact with the components.
![]({{ site.url }}/assets/leveldb-architecture.png)

### Log File
Log file is a on-disk storage to avoid data loss. It is essentially a Write-Ahead Log (WAL). Every time *before* LevelDB is about to write some records, it will write them to a log file first. With this mechanism the data can still be recovered even when system crashes.
    
### Memtables
Memtable is a key in-memory data structure in LevelDB. During a write operation, the newly written records are first stored in Memtable. Once the record size in Memtable hits threshold, LevelDB will create another Immutable Memtable to store the content. Background jobs will be triggered to dump the Immutable Memtable to SSTables for persistence.

### SSTables
[SSTable](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/) is short for *sorted string table*. It contains immutable data stored on disk, and can be easily loaded into memory. It also has indexes to speed up sequential and random read.
Inside SSTables, the keys are sorted in order.

LevelDB dumps Immutable Memtable into an SSTable, creating L0 SSTable. Later with compaction, LevelDB will generate L1, L2 ... L7 SSTable, with higher level SSTables containing older data. Note that the maximum file size for each level is 10X of the previous one.

When a read operation happens, records in Memtable will be searched first, and then search will go down to the different levels of SSTables.

### Manifest & Current
Manifest contains metadata information of SSTables. Basically, for each SSTable, it records the level, smallest key, largest key and file name.

Current file indicates which Manifest file is "current".

## Key Performance Properties
What does the above design buy us? Well, LevelDB is optimized for write performance. The use of Memtable converts DB random write to an in-memory operation. Later, the compaction writes the record in batch sequentially, improving write performance.

The read performance is also decent. sequential reads are fast. Main reason is SSTable. SSTable stores the sorted strings with proper index. It is good for both sequential and random reads. Also, the built-in caching also further improves the read performance. However, when comparing with other read-optimized DB, its read performance may not stand out. In fact, when searching old entries lying in deep layer SSTables, the reads can take longer.

Some benchmark results can be found [here](http://www.lmdb.tech/bench/microbench/benchmark.html). 

## API Design

```C++
// DB creation/deletion.
DB() { };
virtual ~DB();

// DB open
static Status Open(const Options& options,
                   const std::string& name,
                   DB** dbptr);

// Read/Write
virtual Status Put(const WriteOptions& options,
                   const Slice& key,
                   const Slice& value) = 0;
virtual Status Delete(const WriteOptions& options, const Slice& key) = 0;
virtual Status Get(const ReadOptions& options,
                   const Slice& key, std::string* value) = 0;

// Batch operations
virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;

// Iteration
virtual Iterator* NewIterator(const ReadOptions& options) = 0;

// Snapshot
virtual const Snapshot* GetSnapshot() = 0;
virtual void ReleaseSnapshot(const Snapshot* snapshot) = 0;
```

So LevelDB supports three key-value actions - Put, Delete and Get. 
Also, it provides iterators to go over the DB entries. Finally, it also supports snapshot so that the holder can see a consistent view of data. One thing to note is, to signal errors, LevelDB uses a class called Status rather than exception.

## Summary
This post discusses LevelDB components, its user-facing API and some performance properties. In the next series, I will focus on the implementations of its Get/Put operation, and how LevelDB compacts the data at different levels.
