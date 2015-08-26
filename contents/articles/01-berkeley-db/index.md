---
title: Learning about Berkeley DB
author: ksikka
date: 2015-08-26
template: article.jade
---

A review of an article on Berkeley DB, an embedded Key/Value database that's simple, fast, concurrent, and ACID.


## Context

The problem of storing and retrieving data efficiently is ubiquitous in the software industry,
and is one of the primary focuses of the Googles and Facebooks of the software world.
You'd think by now databases are a solved problem, but thriving software products
are constantly struggling to scale their backends to meet growing demand.

My reading queue includes embedded DBs (such as this one),
relational DB servers, various distributed storage/computation systems, and anything else
that made a dent in database history, sorted by chronological order, filtered for interesting-ness.

### The Berkeley DB article from the Architecture of Open Source

I was expecting to learn how BDB worked, but I was surprised to find the article was about architectural decisions and not
a general explanation of how it worked. In different words, the article was about
"how do you draw the lines between modules" rather than "what modules do you need and why".
The authors, Margo Seltzer and Keith Bostic, did a great job
telling stories of how the evolution of BDB revealed the strengths and weaknesses of its architecture,
but they took for granted the reader's fluency in basic database implementation.

The article starts with an Architectural Overview,
which - from my point of view - looked like a conglomeration of boxes labeled `<Noun> Manager`.
My initial impressions:

    Record manager: imagine the owner of a vinyl store
    Log manager: Why? Just use log-rotate
    Lock manager: Why aren't Lock objects enough?
    Buffer manager: What is a buffer in this context?
    Process manager: I thought the OS is supposed to do this

On page 8 of 17, "The Underlying Components" section begins.
The four components underlying the db access methods are
the buffer, lock, log, and transaction managers. I'll attempt to
explain the function of each one in brief.

#### The buffer manager: Mpool

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

#### The lock manager: Lock

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

#### The log manager: Log

The log is a data-structure that sequentially stores the transational operations
that occured, for the purpose of aiding Recovery. If the database crashes
while changes to in-memory buffers have not yet been flushed,
a persisted record of the changes that built up to that point
provides the information necessary to recover. Without the log,
the BDB loses the ability to implement the D in ACID (durability).
More about recovery in the transaction manager section.

The log is implemented on top of Mpool just like a database Btree would be,
which justifies the decision to make Mpool is a separate module.

#### The transaction manager: Txn

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
decisions that BDB made, and I generated a lot of questions.
Writing this forced me to iron out my understanding of the reading,
and a light second-pass proved enlightening.

Out of curiosity, I googled BDB Write ahead log, and stumbled upon
a SQLite page! Turns out SQLite uses a different method of
ensuring durability / recovery by default, but you can turn WAL mode on,
in which case it operates more like BDB. Neat that I can start to
make these connections now. SQLite might make a good next article.
But - I think I'm going to mix it up with a large scale data analysis paper.

Huzzah! We're at the end. I think I will do this again, but
I'm not sure if writing these long write ups is sustainable.
I'll play it by ear. Until next time!
