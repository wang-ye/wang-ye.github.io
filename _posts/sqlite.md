Sqlite Transaction
Sqlite is arguably the most widely used databases in the world.

It has limited transactional support. Let's dive into the source code and see how it is implemented.

ACID property:
1. Atomicity − Ensures that all operations within the work unit are completed successfully; otherwise, the transaction is aborted at the point of failure and previous operations are rolled back to their former state.
2. Consistency − Ensures that the database properly changes states upon a successfully committed transaction.
The consistency property ensures that any transaction will bring the database from one valid state to another. Any data written to the database must be valid according to all defined rules, including constraints, cascades, triggers, and any combination thereof. This does not guarantee correctness of the transaction in all ways the application programmer might have wanted (that is the responsibility of application-level code), but merely that any programming errors cannot result in the violation of any defined rules.

3. Isolation − Enables transactions to operate independently of and transparent to each other.
4. Durability − Ensures that the result or effect of a committed transaction persists in case of a system failure.

Sqlite already has good documentations: https://sqlite.org/lockingv3.html

How?

Why implement locking in pager.c?

Locks

It sets locks on the entire database file.
Sqlite designs its own locking mechnism. The lock has four modes - SHARED, EXCLUSIVE, RESERVED and PENDING, by using native OS system calls.

For SHARED mode, i supports concurrent reads to the DB.
Draw state transition diagram


How is lock implemented?

Isolation is ensured given the locks - when the transaction starts working, the DB snapshot will not change.

Atomicity: either all done all nothing is done.

Journaling for failure recovery:
Data crashing in the middle?

How is Consistency:


<!-- What is the overall archtechure?

What is life of a sqlite DB request?
convert to bytecode, and then execute.

How Indexing implemented?
B-tree

Convert user sql to bytecode? What is a virtual machine?

How sqlite ensure transaction?

Foreign keys?
no support for foreign keys.

B+-- tree

Parsing?

Pager to talk to OS interface and manage the physical pages.


Parsing:

sqlite3.c
static int sqlite3Prepare() {

}

parse.c
sqlite3Parser: the parsing code.

** An instance of this object represents a single SQL statement that
** has been compiled into binary form and is ready to be evaluated.
**
** Think of each SQL statement as a separate computer program.  The
** original SQL text is source code.  A prepared statement object 
** is the compiled object code.  All SQL must be converted into a
** prepared statement before it can be run.
**
** The life-cycle of a prepared statement object usually goes like this:
**
** <ol>
** <li> Create the prepared statement object using [sqlite3_prepare_v2()].
** <li> Bind values to [parameters] using the sqlite3_bind_*()
**      interfaces.
** <li> Run the SQL by calling [sqlite3_step()] one or more times.
** <li> Reset the prepared statement using [sqlite3_reset()] then go back
**      to step 2.  Do this zero or more times.
** <li> Destroy the object using [sqlite3_finalize()].
** </ol>

why locking all the btrees?

sqlite transaction correctness
A:
C:
I:
D:

Relation between pager and b+ tree.
Pager?

What can we learn from sqlite?
test coverage. well tested?

Debugging tips:
https://www.sqlite.org/debugging.html
 -->