A Glance on LevelDB
LevelDB is a well-known open source library for key-value storage.

## Basic Structure
SSTable +  in-memory tables

## What is a SSTable?
Short for sorted string table. Immutable data stored on disk, and can be loaded easily into memory. Metadata about indexes. Good for sequential read.

## Key Performance Properties

Write optimized KV store.
With the use of , writes happens in batch, and  then memory.

Read performance:
Sequential reads are fast. Main reason is SSTable. SSTable is stores the sorted strings and is optimized for sequential read.

Random read access is good, but not so great. When searching old entries lying in deep layers, it can take longer.

## Fault Recovery
It uses a write-ahead logs. For example, if OS system crashes, the WAL will be used to recover the data.

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

## A High-level Diagram

When a new k-v is inserted, it is first written to log. Write ahead logs. Then, it is inserted to Memtable, which is designed to improve random write performance.

## Log Files
What is a log file?

## Compaction
Merge the new data with existing old data in deeper levels.

## Compare with Other KV Store
Snapshot is a very unique thing in leveldb.
How to support a write-heavy use cases

## Putting is All Together - Read & Write

## Snapshot
Why providing snapshot operations? Consistent views of the data. Multi-reading to the same key yields the same results.

## Some Learnings
Extensive use of assertions. Detect data corruption early and stop execution.

Logs

Google C++ coding style - minimalism, simple and clean


## Misc

Reviewing Leveldb
A good series of code analysis in 2013: https://ayende.com/blog/161411/reviewing-leveldb-part-ii-put-some-data-on-the-disk-dude

<!-- ## Iterators
## Version, VersionSet and TableCache
 -->