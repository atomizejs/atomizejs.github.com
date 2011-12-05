---
layout: default
title: API
---

Please also see the [Getting Started](/getting_started.html) page.

# API

## Server

### - `create(server, path)`

* Create a new AtomizeJS server
* Two parameters:
    1. HTTP server: typically returned from `http.createServer()`;
    2. Path within the server to provide the AtomizeJS service
* Note the *client* is available from within the server by accessing
  the `atomize` field

{% highlight javascript %}
var http = require('http');
var atomizeServer = require('atomize-server');
atomizeServer.create(server, '[/]atomize');
server.listen(port, '0.0.0.0');
var atomizeClient = atomizeServer.atomize;
{% endhighlight %}

## Client

Note that with the exception of the constructor, the complete client
API is available in the NodeJS server as shown above under the
`atomizeServer.atomize` field.

### - `Atomize(URL)` constructor

* Browser-side only
* Creates a new atomize client
* Maximum one argument which is the URL to the AtomizeJS server
* If no URL is provided then local-only operation is assumed

{% highlight javascript %}
var atomize = new Atomize("http://localhost:9999/atomize");
{% endhighlight %}

### - `root`

* The `root` object
* All objects and values reachable from the root object are available
  to all clients
* The initial value is the empty object `{}`

### - `atomically(txnFun, contFun)`

* Runs a transaction function
* Two arguments:
    1. The function to be run as a transaction. May return a value.
    2. The function to be run as a continuation once the transaction
    has committed. Will be passed the result of the first argument.

{% highlight javascript %}
atomize.atomically(function () {
    if (atomize.root.number % 2 === 0) {
        return "Even";
    } else {
        return "Odd";
    }
}, function (result) {
    console.log(result);
});
{% endhighlight %}

### - `lift(obj)`

* Prepares an object for management by the AtomizeJS system
* One argument: the object to be managed
* Must be used before assigning an object into another AtomizeJS
  object
* Safe to use on both primitives and objects though is unnecessary for
  primitives
* Idempotent: calling lift on the same object multiple times will
  return the same managed AtomizeJS object

{% highlight javascript %}
atomize.atomically(function () {
    atomize.root.obj = atomize.lift({a: "hello", b: 5});
    atomize.root.obj.c = atomize.lift({});
    atomize.root.obj.b = atomize.lift(6);
}, function () {});
{% endhighlight %}

### - `retry()`

* Indicates the transaction wishes to be suspended pending some change
* No arguments
* Upon retry, the effects of the transaction function are undone
* The transaction will be restarted when the values the transaction
  read prior to the `retry` have been modified

In the following example, the continuation will *only* be invoked with
a value `!== undefined`:

{% highlight javascript %}
atomize.atomically(function () {
    if (atomize.root.value === undefined) {
        atomize.retry();
    } else {
        var result = atomize.root.value;
        delete atomize.root;
        return result;
    }
}, function (result) {
    console.log(result);
});
{% endhighlight %}

### - `orElse(txnFuns, contFun)`

* An abstraction of `retry`
* Two arguments:
    1. A list of transaction functions
    2. A continuation
* The initial *current-transaction-function* is the function at index
  0 of the list
* If the *current-transaction-function* does not `retry` then it
  commits as normal, and the continuation is invoked as normal
* If the *current-transaction-function* hits `retry` then the *writes*
  of the *current-transaction-function* are undone and the
  *current-transaction-function* is now the next transaction function
  in the list. The new *current-transaction-function* is then invoked
* If all transaction functions `retry`, then a normal `retry` is
  issued for *all* objects read across all transaction functions

In the following example, at most one of the transaction functions
will commit. If both retry, then the overall `orElse` operation will
be restarted once any of the objects read by both transaction
functions have changed (in this case, the complete *read set* is
`atomize.root`, `atomize.root.a`, `atomize.root.b`; please see the
[background](/background.html) page for more details on *read sets*).

{% highlight javascript %}
atomize.orElse(
    [function () {
         if (atomize.root.a.toIncrement === undefined) {
             atomize.retry();
         } else {
             atomize.root.a.toIncrement += 1;
             return atomize.root.a.toIncrement;
         }
     },
     function () {
         if (atomize.root.b.toDecrement === undefined) {
             atomize.retry();
         } else {
             atomize.root.b.toDecrement -= 1;
             return atomize.root.b.toDecrement;
         }
     }], function (result) {
         console.log(result);
     });
{% endhighlight %}
