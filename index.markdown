---
title: AtomizeJS
---

# AtomizeJS

*A Distributed Software Transactional Memory implementation in
JavaScript.*

## Introduction

Writing concurrent programs is tricky. Languages tend to go down one
of two roots: either threads are allowed to directly access the same
data-structures, and it's left up to the programmer to decide how to
manage locking; or the language presents a model whereby each thread
has its own specific memory, and data is passed between threads
through message passing. The former approach gives the programmer a
slightly bigger gun to shoot themselves with: the ease of writing code
that deadlocks is unrivalled. The latter is conceptually much simpler
and more intuitive, but is relatively confined: languages such as
Erlang or dedicated actor frameworks such as Akka are about as
mainstream as it gets.

Writing distributed programs is trickier still. In addition to the
issues of writing concurrent programs, you have further difficulties
of dealing with the safe access or distribution of data across the
system, combined with the increased potential for partial failures.

AtomizeJS is a project which aims to make it easy to write programs in
JavaScript that can be both concurrent and distributed. It does this
by implementing Distributed Software Transactional Memory.

The mental model you should have when using AtomizeJS is as follows:

* Assume there is an object graph that gets distributed automatically
  to every browser looking at your site.
* To make safe changes to this object graph, you write functions which
  change the objects as desired. These functions are then run by
  AtomizeJS as transactions. AtomizeJS ensures that these transaction
  functions are run:
    * *atomically*: Either the transaction functions completes or
      fails in its entirety - it is impossible for a transaction to
      partly succeed.
    * *consistently*: The transaction function starts with a
      consistent view of the object graph, and it ends with a
      consistent view. If the transaction succeeds then the changes
      the transaction function has made are completely applied. If the
      transaction errors, then any changes the transaction function
      has made are completely undone.
    * *in isolation*: The transaction function is run as if no other
      transactions are running at the same time - the transaction
      function does not have to think about locking or critical
      regions.

Transactions cannot deadlock: when a transaction function is run,
AtomizeJS detects if other transactions have modified the same objects
that are being altered by this function. If so, this function is
automatically restarted with the updated current state-of-the-world.

Transactions allow programmers to safely manipulate data-structures
shared between many threads, or in the case of AtomizeJS, clients
(such as web browsers). You don't need to write distribution
mechanisms as you would with message-passing: all the code you write
assumes the data is local - you never write `send` or have `receive`
call-backs.
