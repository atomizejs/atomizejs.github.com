---
layout: default
title: Background
---

# Software Transactional Memory

## Background

[Software Transactional Memory](http://en.wikipedia.org/wiki/Software_transactional_memory)
(STM) was
[first proposed](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.41.4743)
by Nir Shavit and Dan Touitou in 1995. Following this, academic
interest in transactional memory increased substantially, and a large
number of papers followed, covering different approaches, languages,
and implementation choices. Interest peaked in the early 2000s, and
Microsoft worked hard on adding STM support throughout the .NET
framework, only to
[later abandon it](http://www.bluebytesoftware.com/blog/2010/01/03/ABriefRetrospectiveOnTransactionalMemory.aspx). There
are currently actively used and developed implementations of STM for
the [C++ Boost library](http://eces.colorado.edu/~gottschl/dracoSTM/)
and for
[Haskell](http://research.microsoft.com/pubs/67418/2005-ppopp-composable.pdf)
amongst others.

STM is primarily aimed at trying to ease the difficulties of writing
large-scale concurrent programs. In concurrent programs, where
concurrent access is required to the same data-structures, locking is
typically used to ensure multiple threads can't proceed through
*critical regions* concurrently. If multiple threads were to progress
through *critical regions* at the same time, they may be able to see
the partial progress and intermediate states caused by other
threads. Thus the locks are used to ensure the *atomic*, *consistent*
and *isolated* properties.

Manual locks are notorious for being difficult to get right. If you
use many locks (fine-grained locking) where each lock is used to
protect a small data-structure then obtaining all the locks in such a
way as to avoid potentially causing a deadlock can be tricky. If you
have few locks (coarse-grained locking) then whilst it's much easier
to avoid deadlock, you run the risk of losing available parallelism
because each lock protects too large a data-structure.

STM avoids the programmer having to take any locks explicitly at
all. Instead, in a transaction, the STM engine is either directly told
which data-structures are being accessed (Boost.STM does this) or is
able to infer it (either by the type-system, as with the Haskell
implementation; or at run-time by proxies and other techniques, as
with AtomizeJS). Dereferencing counts as a *read* of an object (so
`a.x` is a *read* of object `a`); assignment counts as a *write* (so
`a.x = y` is a *write* to object `a`). For example:

{% highlight javascript %}
if (root.calendar.dayOfMonth === 1 && root.calendar.month === "January") {
    root.global.login.motd = "Happy New Year!";
}
{% endhighlight %}

Here, we have a *read set* of at least `root` and `root.calendar`. If
it did happen to be the 1st of January, then we would further *add* to
the *read set* `root.global`, and we would have a *write set* of
`root.global.login`. Note that we do not have
`root.calendar.dayOfMonth` in the *read set*: we did not try and
inspect any children of `dayOfMonth`. (Equally, if you assign to
`root.calendar.dayOfMonth` then the *write set* would include
`root.calendar`, not `root.calendar.dayOfMonth`, and so the reads and
writes match.)

## AtomizeJS

There are then several ways to implement STM. What follows is an
outline of how AtomizeJS works. This is not dissimilar to other
implementations.

AtomizeJS keeps a *transaction log*. This contains all the reads and
writes performed in the current transaction. Writes in the current
transaction are *not* performed against the real object, but merely
cached in the transaction log, and then reads must first check the
transaction log in order to ensure the correct value is returned. This
approach means that if the transaction needs to be abandoned, the
transaction log is just thrown away and nothing else needs doing.

When the transaction function completes, the transaction log contains
the net effect of the transaction function. This log needs to be
verified and committed. Every object is versioned, and the log
captures the version of every object that has been read or written
to. This information is then passed to the server which tests to see
whether the server-copy of those objects is the same version as the
client used, or whether some other client has modified any of the
objects in the meantime, and thus changed the version numbers. If
there are any discrepancies between the version numbers in the
transaction log and on the server then the client is sent updates for
the objects for which it holds out-of-date copies, and is then told to
restart the transaction. Eventually, the transaction will be performed
against the current values of all relevant objects, the server will
verify this and will then apply the writes from the transaction log to
its copies of those objects, and bump their version numbers. It will
then confirm the commit back to the client which will similarly apply
the writes to the real objects and bump those version
numbers. Finally, the client will invoke the transaction function
continuation.

Currently, because the server is implemented simply in NodeJS, there
is no concurrency on the server. As such, no locks need to be
taken. However, with a more sophisticated concurrent implementation,
during the commit on the server, locks on every object read or written
to would need to be taken.

## Limitations

The key attraction of STM is that you don't need to think about which
locks to take: you can just write a function, pass it to `atomically`
and expect it to be run with all the correct properties. Provided the
function *only* accesses and manipulates data-structures which are
controlled by the STM engine, this is indeed true: the transaction log
will capture all reads and writes as required, and the transaction
will be managed correctly.

However, if the transaction function manipulates objects outside of
STM engine, or performs other operations which have side effects (such
as as disk access, or network access, or launching the missiles) then
there's really no way for the transaction log to capture and simulate
the effect of this. For example, consider a function which tries to
send a network packet to a remote host, and receive a response from
that remote host. Doing this in a transaction is nonsensical: if the
transaction throws an exception and has to abort, how do you undo the
send, or the receive?

Thus STM does not make the impossible possible: without making the
entire universe transactional, it would not be possible to achieve
such things. It would seem that STM was billed to eventually be able
to solve such problems, but inevitably was unable to. This is one of
the reasons, amongst others, that
[Microsoft abandoned its STM in .NET](http://www.bluebytesoftware.com/blog/2010/01/03/ABriefRetrospectiveOnTransactionalMemory.aspx).

However, when these limitations are understood (and indeed, if you
think about what you would expect to be possible if you were manually
using locks then it's all rather obvious), STM is a very useful and
valuable technology that does make writing concurrent programs easier:
the STM engine is able to detect collisions at the finest-grained
level and, experiments have shown that with concurrent implementations
of the engine, is able to perform extremely well. In the case of
AtomizeJS, whilst the current server is not multi-threaded, clients
still perform the transaction functions in parallel before sending the
transaction log to the server. Thus it's still possible to see how
parallel execution can occur.

We believe the AtomizeJS STM engine, in combination with the
distribution of objects to clients in a safe way (i.e. respecting the
*atomic*, *consistent* and *isolated* properties) creates a powerful
mechanism for writing sophisticated applications in JavaScript.
