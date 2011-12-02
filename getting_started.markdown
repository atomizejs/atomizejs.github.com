---
layout: default
title: Getting Started
---

# Server

For these first examples, we shall write no server code and just use a
plain AtomizeJS server. See the [install](/install.html) guide for
acquiring the libraries. Once you've done that, you just need to put
the following boiler-plate in a file (call it `app.js`):

{% highlight javascript %}
var http = require('http');
var atomizeServer = require('atomize-server');
var server = http.createServer();
var port = 9999;
var port_index = process.argv.indexOf('--port');
if (port_index > -1) {
    port = process.argv[port_index + 1];
}

atomizeServer.create(server, '[/]atomize');
console.log(" [*] Listening on 0.0.0.0:" + port);
server.listen(port, '0.0.0.0');
{% endhighlight %}

Then start up NodeJS with
    node --harmony-collections --harmony-proxies app.js

This will create a server listening on port 9999 that will provide
access to the AtomizeJS system under the path `/atomize`.


# Client

In the following examples, when we wish to output something, we use
`console.log` which will mean you'll need to be watching the
JavaScript console in your web browser. Obviously, you can rework
these to achieve fancy interactions with the DOM if you so wish.

All these examples assume the following HTML template:

{% highlight html %}
<!DOCTYPE html>
<html>
<head>
<title>AtomizeJS Example</title>
<script src="http://cdn.sockjs.org/sockjs-0.1.min.js"></script>
<script type="text/javascript" src="https://raw.github.com/atomizejs/cereal/master/lib/cereal.js"></script>
<script type="text/javascript" src="https://raw.github.com/atomizejs/atomize-client/master/lib/atomize.js"></script>
<script>
var atomize = new Atomize("http://localhost:9999/atomize");

// rest of example goes here

</script>
</head>
<body onload="start();">
</body>
</html>
{% endhighlight %}

As you can see, this assumes that the AtomizeJS server is running on
`localhost` and listening on port 9999. If this is not the case, then
please adjust this template as required.

None of these examples need to be *served* by a web-server: provided
you have the AtomizeJS server running as described above, pointing
your browser directly at the HTML files directly on your machine
(i.e. with a `file:///...` URL) will work perfectly well.

# Writing and reading

{% highlight javascript %}
function start () {
    atomize.atomically(function () {
        if (atomize.root.x === undefined) {
            atomize.root.x = Date.toString();
            return "Wrote " + atomize.root.x;
        } else {
            var result = atomize.root.x;
            delete atomize.root.x;
            return "Read " + result;
        }
    }, function (result) {
        console.log(result);
    });
}
{% endhighlight %}

This introduces quite a lot of concepts.

1. The `atomize.atomically` function takes two arguments.
    1. The first argument is the function that is run as a
      transaction, atomically. This function may run more than once:
      if it is found to have read or written data which has in the
      mean time been modified by another transaction, then the effects
      of this function will be undone, and the transaction is
      restarted automatically. Any result that this function returns
      is passed to the second argument.
    2. The second argument is a continuation function. It is run once
      the transaction successfully commits, and is passed any result
      returned from the transaction function. This function will only
      be run at most once.
2. `atomize.root` is the root of the distributed object
  graph. Assigning values to properties and objects beneath the
  `atomize.root` object will ensure that those values are distributed
  to all clients using this AtomizeJS server. The initial value of the
  `atomize.root` object is the empty object, `{}`.
3. The same transaction has different behaviours depending on what it
  finds. If the client is the first client there, then it will find
  that the `x` property is `undefined`. It will then assign to the `x`
  property, and return, thus completing the transaction. The second
  client will find that the `x` property is defined, and so will go
  into the other branch of the `if` statement.
4. State continues to exist in the server even if clients leave. If
  you start the AtomizeJS server, then point a client at this HTML
  page, that client will be the first client there, and will write the
  current date to the `x` property of the `root` object. The effect of
  this transaction exists on the server. Thus even if you shut your
  browser down and restart it, when you next visit the page, you'll be
  the second client there, and thus will read the previously written
  date.

## The importance of continuations

It is easy to forget that the continuation-passing-style model is
required. For example, you might be tempted to write the above code
as:

{% highlight javascript %}
function start () {
    var result;
    atomize.atomically(function () {
        if (atomize.root.x === undefined) {
            atomize.root.x = Date.toString();
            result = "Wrote " + atomize.root.x;
        } else {
            var result = atomize.root.x;
            delete atomize.root.x;
            result = "Read " + result;
        }
    });
    console.log(result);
}
{% endhighlight %}

If you do this, what you'll most likely see is that every browser that
visits the HTML page will log to its console `Wrote ...` and the
date. This is because whilst the first client really will write the
date to the `x` property, the second client will start the transaction
with the default empty root object, and thus start down the first
branch of the `if` statement. It is only when the second client tries
to commit the effects of the transaction that it'll find out from the
AtomizeJS server that it ran the transaction function on out-of-date
data. At that point, the second client will be sent the most
up-to-date version of the `root` object, which will have the `x`
property set, and the transaction function will be restarted.

But the communication from the client to the AtomizeJS server is
asynchronous, which means that the transaction function will return
and execution will continue to the `console.log` line whilst the
transaction is trying to commit for the first time (which will likely
fail). Thus even though the transaction function will be rerun, and
even though eventually the `result` variable will be set to the
correct value, the `console.log` line will have been run too early,
and the wrong result output because at that point in time, the
transaction function had gone down the first branch, and thus assigned
the `Wrote ...` result to the `result` variable. This is why it's
important to use the continuation-passing-style whenever you need to
depend on the transaction having committed successfully.

## Avoiding side effects

Equally, it's important to avoid doing things within a transaction
which have *side-effects*. For example, if you rewrote this code as:

{% highlight javascript %}
function start () {
    atomize.atomically(function () {
        if (atomize.root.x === undefined) {
            atomize.root.x = Date.toString();
            console.log("Wrote " + atomize.root.x);
        } else {
            var result = atomize.root.x;
            delete atomize.root.x;
            console.log("Read " + result);
        }
    });
}
{% endhighlight %}

then on the second client to visit the page, you'll probably see two
lines output on the console: the first being `Wrote ...` and the
second being `Read ...`. The reason is the same as described above
(i.e. the second client has to rerun the transaction function as the
commit will fail after the first run), but now, because we're doing
the `console.log` *within* the transaction function, the `console.log`
will occur every time the transaction function is run.

Whilst a transaction that fails to commit undoes all the modifications
of objects managed by AtomizeJS (including everything that is or has
been reachable from `atomize.root`), it can't undo other actions. Thus
modifications of the DOM and other operations that cause side-effects
should only be done from the continuation to the transaction
function. Thus it's best practise to return values from the
transaction that allow the continuation to figure out what it should
be doing. The continuation is only run once, and only run once the
transaction has successfully committed, so by that point, the result
of the transaction function is stable.

## Writing Objects and Primitives

So far, we have written only primitive values into the `root` object
graph. When you assign a primitive value (i.e. a boolean, a string, a
number or `undefined`), you don't need to do anything
special. However, if you assign an object, then you need to `lift` the
object to ensure that AtomizeJS starts managing the object
correctly. For example:

{% highlight javascript %}
function start () {
    atomize.atomically(function () {
        if (atomize.root.x === undefined) {
            atomize.root.x = atomize.lift({a: "hello"});
            atomize.root.x.date = Date.toString();
            return "Wrote " + atomize.root.x.date;
        } else {
            var result = atomize.root.x.date;
            delete atomize.root.x;
            return "Read " + result;
        }
    }, function (result) {
        console.log(result);
    });
}
{% endhighlight %}

If you do not call `lift` then you'll get an error.

## Objects outside of `root`

Any call to `lift` will return an object that is managed by
AtomizeJS. This need not be part of the `root` object. For example:

{% highlight javascript %}
var myObj;
function start () {
    atomize.atomically(function () {
        return atomize.lift({});
    }, function (obj) {
        myObj = obj;
    });
}
{% endhighlight %}

`myObj` now contains an object that is managed by AtomizeJS. Of
course, because it's not reachable from the `root` object, it won't
get distributed to other clients. But, for example, we can take
objects that were reachable from the `root` object, and hold on to
them, even if later on, they get detached from the `root` object. We
can use this to build a broadcast queue...


# Building communication

So far, we've seen how you can build transactions that act on what
they find in the `root` object, and can modify properties and values
within the `root` object. For communication between different parties,
you need *sending* operations and *receiving* operations. In our case,
*sending* is very simple: it's just writing to a known location within
the `root` object. For *receiving*, we'd like to avoid *spinning*: if
no new value has been written for us to receive, we don't want to be
constantly restarting the transaction. Instead, we want the AtomizeJS
system to remember the transaction, and automatically restart it when
the values of variables we've read so far in the transaction have been
changed. This avoids the need for spinning. This is the purpose of the
`retry` operation. For example:

{% highlight javascript %}
function start () {
    if (Math.random() > 0.2) {
        receive();
    } else {
        send();
    }
}

function receive () {
    atomize.atomically(function () {
        if (atomize.root.value === undefined) {
            atomize.retry();
        } else {
            return "Received " + atomize.root.value;
        }
    }, function (result) {
        console.log(result);
    });
}

function send () {
    atomize.atomically(function () {
        atomize.root.value = Date().toString();
    });
}
{% endhighlight %}

Try opening up several browser windows and pointing them all at the
same HTML page.

Here, we have an 80% chance that a new client will enter the `receive`
function. If there is no value to receive then it will be suspended,
and only woken up and restarted once the variables that it has read so
far (in this case, just the `root` object) are modified.

Eventually, one of the clients will enter the `send` function, and
will write to the `value` property of the `root` object. At this
point, all the other clients, waiting in the `receive` transaction,
will have their transaction function restarted. The second time
through this receive transaction, the clients will see the newly
written `value` property. This time, they'll not go into the `retry`
branch of the `if` statement, and instead will receive the result and
complete the transaction. We've just built a broadcast mechanism.

Whilst the `retry` operation suspends the current transaction, it too
does not block. Thus as before, it's important to use
continuation-passing-style when you need to run code after the
transaction has finally completed.

## A broadcast queue

This example "puts it all together". It is a broadcast queue with one
writer which constantly adds new values to the end of the queue. Every
reader will receive every value that was added *after* the client
connected. All clients can work at their own pace.

This example makes use of `retry` and the fact that you can take
objects out of the `root` object graph and they're still managed by
AtomizeJS: this is how each client maintains its own position into the
queue.

The queue that we construct is going to be a standard
linked-list. Each *cell* has a property `value` (containing the value
of the cell), and a property `next` (which points to the next
cell). The `root.queue` pointer is always going to be the *tail* of
the queue, where new values get added.

First, we need to ensure we only have one writer:

{% highlight javascript %}
var writer;
var nextValue = 0;

function start() {
    atomize.atomically(function () {
        if (atomize.root.queue === undefined) {
            atomize.root.queue = atomize.lift({
                    next: atomize.lift({}),
                    value: nextValue
                });
            return true;
        } else {
            return false;
        }
    }, function (w) {
        writer = w;
        loop();
    });
}
{% endhighlight %}

By the end of this, the variable `writer` will contain `true` for just
one client -- the client that arrived first. Now we need to set up the
loop. We want to make sure this doesn't block the browser so we don't
use a `while` loop or recursion. Instead, we use `setTimeout`:

{% highlight javascript %}
function loop() {
    if (writer) {
        setTimeout("write();", 1);
    } else {
        setTimeout("read();", 1);
    }
}
{% endhighlight %}

Now to write to the queue:

{% highlight javascript %}
function write() {
    nextValue += 1;
    atomize.atomically(function () {
        var cell = atomize.root.queue.next;
        cell.next = atomize.lift({});
        cell.value = nextValue;
        atomize.root.queue = cell;
        return cell.value;
    }, function (value) {
        console.log("Wrote " + value);
        loop();
    });
}
{% endhighlight %}

The *tail* of the queue will always contain a pointer to the `next`
cell which will be empty. Thus we take the `next` cell, and populate
it with our new value (and new empty cell), and then assign that back
to the `queue`, thus adding to the tail of the queue. Once the `write`
transaction has completed, we loop back around to write the next
value.

Now for reading:

{% highlight javascript %}
var myPosition;

function read() {
    if (myPosition === undefined) {
        atomize.atomically(function () {
            if (myPosition === undefined) {
                myPosition = atomize.root.queue;
            }
        }, function () {
            read();
        });
    } else {
        atomize.atomically(function () {
            if (myPosition.value === undefined) {
                atomize.retry();
            } else {
                return {value: myPosition.value, next: myPosition.next};
            }
        }, function (result) {
            myPosition = result.next;
            console.log("Read " + result.value);
            loop();
        });
    }
}
{% endhighlight %}

First, we have to populate the `myPosition` variable with the cell at
the current tail of the queue, which we do in a separate
transaction. Once we have a cell in `myPosition` we wait for it to
have a `value` property, and once it does, we grab that value, and
then update our position to the next cell in the linked list. This
way, we don't rely on keeping up with the writer. The value of
`myPosition` is an object that is managed by AtomizeJS: it was created
by a call to `lift` in `write`. If the writer goes faster than the
reader then the value of `myPosition` will differ from
`atomize.root.queue`, but that's ok: the reader will always be able to
follow the links and walk the chain to the tail of the queue.
