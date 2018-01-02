
Records enables users to write and execute plain SQL directly. It supports various DB formats including MySql, Sqlite, Postgres and Oracle.

For example, 

```python
import records

db = records.Database('postgres://...')
rows = db.query('select * from active_users')    # or db.query_file('sqls/active-users.sql')
```


It is a thin wrapper on top of [Sqlalchemy](http://www.sqlalchemy.org/). Sqlalchemy returns a list of DB records, then *Records* converts it to a RecordCollection object, which is essentially a list of *Record* object. In this post, we will discuss some implementation details of this utility.

## Defining Record Class
Record class is the basic unit to store the DB query results. The definition is as follows:

```python
class Record(object):
    """A row, from a query, from a database."""

    # Slot for memory saving.
    __slots__ = ('_keys', '_values')

    def __init__(self, keys, values):
        self._keys = keys
        self._values = values

        # Ensure that lengths match properly.
        assert len(self._keys) == len(self._values)

    def keys(self):
        """Returns the list of column names from the query."""
        return self._keys

    def values(self):
        """Returns the list of values from the query."""
        return self._values   
```

As we can see, Record uses slot to store the attributes. We have seen the usage before in (). Slots can be great when the attributes are fixed and many objects of the class will be initialized.

```python
class Record(object):
    def __getitem__(self, key):
        # Support for index-based lookup.
        if isinstance(key, int):
            return self.values()[key]

        # Support for string-based lookup.
        if key in self.keys():
            i = self.keys().index(key)
            if self.keys().count(key) > 1:
                raise KeyError("Record contains multiple '{}' fields.".format(key))
            return self.values()[i]

        raise KeyError("Record contains no '{}' field.".format(key))
```

The *__getitem__* method provides easier access to the Record data. It supports both index- and string- based access to the record.

## The Database Abstraction

```python
class Database(object):
    """A Database connection."""

    def __init__(self, db_url=None, **kwargs):
        # If no db_url was provided, fallback to $DATABASE_URL.
        self.db_url = db_url or DATABASE_URL

        if not self.db_url:
            raise ValueError('You must provide a db_url.')

        # create_engine is imported from sqlalchemy.
        self._engine = create_engine(self.db_url, **kwargs)

        # Connect to the database.
        self.db = self._engine.connect()
        self.open = True

    ...

    def query(self, query, fetchall=False, **params):
        """Executes the given SQL query against the Database. Parameters
        can, optionally, be provided. Returns a RecordCollection, which can be
        iterated over to get result rows as dictionaries.
        """

        # Execute the given query.
        cursor = self.db.execute(text(query), **params)  # TODO: PARAMS GO HERE

        # Row-by-row Record generator.
        row_gen = (Record(cursor.keys(), row) for row in cursor)

        # Convert psycopg2 results to RecordCollection.
        results = RecordCollection(row_gen)

        # Fetch all results if desired.
        if fetchall:
            results.all()

        return results

    def transaction(self):
        """Returns a transaction object. Call ``commit`` or ``rollback``
        on the returned object as appropriate."""
        return self.db.begin()
```

For Database object, we will focus on two aspects: how to query DB and handle transaction.

DB query:
It utilizes the sqlalchemy execute approach, and returns a cursor object. Then, it constructs a RecordCollection with the generator for the row data.

For transaction, it defers to the lower level sqlalchemy to handle the transaction.
Transactions: t = Database.transaction(); t.commit().


## What is RecordCollection?

```python
class RecordCollection(object):
    """A set of excellent Records from a query."""

    def __init__(self, rows):
        self._rows = rows
        self._all_rows = []
        self.pending = True

    def __iter__(self):
        """Iterate over all rows, consuming the underlying generator only when necessary."""
        i = 0
        while True:
            # Other code may have iterated between yields,
            # so always check the cache.
            if i < len(self):
                yield self[i]
            else:
                # Throws StopIteration when done.
                # Prevent StopIteration bubbling from generator, following https://www.python.org/dev/peps/pep-0479/
                try:
                    yield next(self)
                except StopIteration:
                    return
            i += 1

    def next(self):
        return self.__next__()

    def __next__(self):
        try:
            nextrow = next(self._rows)
            self._all_rows.append(nextrow)
            return nextrow
        except StopIteration:
            self.pending = False
            raise StopIteration('RecordCollection contains no more rows.')

    def all(self, as_dict=False, as_ordereddict=False):
        """Returns a list of all rows for the RecordCollection.
        If they haven't been fetched yet, consume the iterator and cache the results.
        """

        # By calling list it calls the __iter__ method
        rows = list(self)

        if as_dict:
            return [r.as_dict() for r in rows]
        elif as_ordereddict:
            return [r.as_dict(ordered=True) for r in rows]

        return rows

    def __getitem__(self, key):
        is_int = isinstance(key, int)

        # Convert RecordCollection[1] into slice.
        if is_int:
            key = slice(key, key + 1)

        while len(self) < key.stop or key.stop is None:
            try:
                next(self)
            except StopIteration:
                break

        rows = self._all_rows[key]
        if is_int:
            return rows[0]
        else:
            return RecordCollection(iter(rows))
```

RecordCollection is an iterator, which wraps the underlying cursor generator. As an iterator, it must implement the __iter__ method. The tricky thing is the records are cached with *self._all_rows*.

### First method

### All method

## Summary
It provides a good example for implementing many magic methods, such as __iter__, __getitem__ __getattr__  __len__ __dir__ __repr__
