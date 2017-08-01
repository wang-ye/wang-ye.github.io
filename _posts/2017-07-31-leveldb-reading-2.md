
## Key Operations

As a key-value store, leveldb supports read and write. What is unique about leveldb is it supports versioning and snapshots.

## Leveldb Read
Given the physical layout, there can be duplicate keys appearing in different levels. The right semantics is reading the latest entry, i.e., Memtable > Immutable Memtable > L0 SSTable > L1 SSTable > ... > L7 SSTable.

## Leveldb Write

```C++
// Default implementations of convenience methods that subclasses of DB
// can call if they wish
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  batch.Put(key, value);
  return Write(opt, &batch);
}
```

### Compaction
Merge the new data with existing old data in deeper levels.

## Compare with Other KV Store
Snapshot is a very unique thing in leveldb.
How to support a write-heavy use cases

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