Sqlite Source Reading

What is the overall archtechure?

What is life of a sqlite DB request?
convert to bytecode, and then execute.

How Indexing implemented?
B-tree

Convert user sql to bytecode? What is a virtual machine?


ACID? Transaction?
locks

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
