---
title: Berkeley DB
author: ksikka
date: 2015-08-26
template: article.jade
---

Berkeley DB is an open-source Key/Value database that lives in
your address space than running as a separate process, so it has a light footprint.
It's optimized for speed and supports advanced features like concurrency,
ACID transactions, and replication. The interface to it is way simpler than SQL, since it's
not relational. It's evolved for over 20 years now.

## Why

I read about Berkeley DB first because of its simplicity and historical importance.
BDB solves a narrower problem than a modern RDBMS.
It was written in 1970 and still used today, so I'm sure modern databases today
draw a lot from it.

## How

I read the Berkeley DB article from The Architecture of Open Source Applications.

Starting off, the author got me excited to learn how BDB worked,
but then dove into architectural discussion without framing the discussion well.
Later on, the author discusses the individual modules and then the beginning
starts to make much more sense.

I remember looking at the Architecture Diagrams thinking:

1. **Record manager:** Imagine the owner of a vinyl store
2. **Log manager:** Just use log-rotate
3. **Lock manager:** Aren't simple locks enough?
4. **Buffer manager:** This means nothing to me

A few pages later, and I got to the good part: The Underlying Modules.
I'll explain each one in brief.

#### 1. The buffer manager: Mpool

Where the BTrees live and flourish. Well, that's what the rest of BDB thinks.

BDB data structures are not directly implemented on memory,
nor are they directly implemented on files. The data structure "pointers"
are not memory addresses; rather, they're page numbers (and maybe an offset into the page?).
A "page" is a discrete chunk of memory that you can request from Mpool.
There's probably some limit to the number of pages you can be operating on at one time,
but there's virtually no limit on the number of pages you can have total,
allowing the total database size to be greater than the amount of available memory.
Pages live on disk, and when you request a page, Mpool reads it from disk
and caches the page. Future writes occur in memory only until you call "flush" on the page.
Pages can be pinned to memory to prevent eviction, and unpinned to allow cache eviction.

Mutating data structures in memory means that there is risk of data loss
if the process crashes before the page is flushed.
However, if you turn on Write Ahead Logging, BDB will write and persist
a log entry before the operation occurs.
This feature does hurt performance, since the log page will need to be flushed on every operation.
The Log and Transaction sections explain this in more detail.

#### 2. The lock manager: Lock

Logic to avoid stepping on toes. More like a module than a manager if you ask me.

BDB uses page-level locking to ensure atomic operations. I infer this to mean
that when you're mutating a data structure on a page, you acquire a lock so that
others must wait till you're done writing to read the page. Similarly, if you're reading
the page, you want to enforce that people can't write to the page until you're done reading.
However, if you two operations want to read from a page, they may do so concurrently.
The rules for which locks conflict are encoded in a "Conflict Matrix".

The conflict matrix enables Lock to support Hierarchical Locking.
I didn't take the time to understand this fully, but what shocked me was that
the authors of BDB went through the trouble to support this,
yet "Berkeley DB doesn't use hierarchical locking internally" (straight from the article).

A more practical justification for the Conflict Matrix is that it separates
concerns. Locking code doesn't need to be concerned with the specific lock configuration.
And describing the lock configuration shouldn't be intertwined with core locking code.
Separating the two makes it easier to change the configuration without worrying
about introducing nasty bugs. It also makes it easier to test the core Lock logic.

#### 3. The log manager: Log

Powering "breadcrumbs", like the ones in Hansel and Gretel.

The log is a data-structure that sequentially stores the transational operations
that occured, for the purpose of aiding Recovery. If the database crashes
while changes to in-memory buffers have not yet been flushed,
a persisted record of the changes that built up to that point
provides the information necessary to recover. Without the log,
the BDB loses the ability to implement the D in ACID (durability).
More about recovery in the transaction manager section.

The log is implemented on top of Mpool just like a database Btree would be,
which justifies the decision to make Mpool is a separate module.

#### 4. The transaction manager: Txn

Dropping the "breadcrumbs" and following them back (and more not covered here).

Transactions are what allow BDB support ACID. Full ACID transaction
support is optional in BDB - you can turn it off for faster performance,
or on for greater ACIDity. When you turn transactions on, every DB query
is automatically wrapped in a transaction, and as mentioned before,
BDB writes a durable log entry before a transaction begins.
Writing to the log in a persistent fashion is one of the reasons
for the ACID/performance tradeoff.

There's plenty more to learn about how BDB handles transactions, but
the author chose to highlight Recovery as one worth discussing.
Txn is responsible for taking checkpoints - though it's not clear
from the article on what schedule. Every so often? After some number
of txns? Taking a checkpoint means forcing all dirty Mpool buffers
to flush and writing a checkpoint log record that says
"the pages on disk accurately reflect all the txns up till now".

Recovery is a conceptually simple operation. From the last txn log record,
go backwards in the log "undoing" any txns which were BEGIN but not COMMIT.
This rectifies the state for all uncommitted or aborted transactions.
Then go forward from the checkpoint to the end of the log, "doing" any txns
which were committed. Now the state is correct. Take a checkpoint
to prevent this work from happening again.

It bothers me that I don't fully understand the implementation of "undoing"
and "doing" above. I'm not sure how BDB would handle random data corruptions.

## So much to learn

For a first pass, this was fruitful. I learned a lot about the general
decisions that BDB made, and it generated a lot of questions.

Out of curiosity, I googled "BDB Write ahead log", and stumbled upon
a SQLite page! It turns out SQLite uses a different method of
recovery, but supports WAL optionally. I'll read about that sometime,
but next up on my queue is A Comparison of Approaches to Large Scale Data Analysis.

Huzzah! We're at the end. I think I will do this again, but
I'm not sure if writing write-ups this long is sustainable.
I hope it gets easier with practice. I'll play it by ear. Until next time!
