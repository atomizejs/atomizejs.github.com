---
layout: post
title: Starting to decloak
---

Under 1 man-month of development done; AtomizeJS is public, but there
is much to do. The most major limitation is that of browser support:
currently only Firefox 8 and Chrome 17 are supported. This is because
they support experimental features of the next version of JavaScript
which we depend upon. We have some ideas about how to support other
browsers and doing this is our first priority, but it's not the only
thing on our to-do list.

There is also much work to be done on:

* Exception handling: particularly transactions that throw exceptions
  after they've been restarted. Having a good story on dealing with
  failure scenarios is very important.

* Garbage Collection: currently, any object that has been lifted into
  AtomizeJS will exist on the server forever. Distributed GC is on the
  whole quite a challenge. I have some ideas how to solve this, but
  it's not a straight-forward problem.

* Multi-node server support: it's pretty simple to imagine having
  multiple NodeJS servers all attached to the same AtomizeJS
  instance. This should be fairly simple to achieve: the lack of a
  SockJS client for NodeJS is the reason it doesn't work yet, but I
  could do a plain socket implementation.

* Presence: it's fairly simple to build a system where a new client
  makes its presence known to the other clients, though there are some
  open questions about naming clients. It's much harder currently for
  other clients to know about the loss of a client. Indeed, quite what
  loss means may vary from application to application. Having some
  mechanism for being able to indicate which clients currently exist,
  for varying degrees of exist, would be useful for a large number of
  applications.

* Security and partitioning: the focus of AtomizeJS is to make it
  easier to move more and more logic to the browser side. However,
  there are always going to be applications which need to have some
  server-side component. One of the likely important areas here is to
  provide a means whereby the server can control which clients can
  read and write to which variables. Currently there is no security at
  all: any client can write and read to and from any object managed by
  AtomizeJS (provided they can get hold of it in the first place -
  objects do not have to be reachable via `root`). It's easy to
  imagine needing private objects amongst different clients and the
  server.

* Optimisations: There are many optimisations that could be done both
  client and server side.
