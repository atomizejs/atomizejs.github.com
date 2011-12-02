---
layout: default
title: Getting Started
---

# Server

For these first examples, we shall write no server code and just use a
plain AtomizeJS server. See the [install](/install.html) guide for
acquiring the libraries. Once you've done that, you just need to put
the following boiler-plate in a file (call it `app.js`):

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

Then start up NodeJS with
    node --harmony-collections --harmony-proxies app.js

This will create a server listening on port 9999 that will provide
access to the AtomizeJS system under the path `/atomize`.


# Client

In the following examples, when we wish to output something, we use
`console.log` which will mean you'll need to be watching the
JavaScript console in your web browser. Obviously, you can rework
these do achieve fancy interactions with the DOM if you so wish.

All these examples assume the following HTML template:

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

As you can see, this assumes that the AtomizeJS server is running on
`localhost` and listening on port 9999. If this is not the case, then
please adjust this template as required.

None of these examples need to be *served* by a web-server: provided
you have the AtomizeJS server running as described above, pointing
your browser directly at the HTML files directly on your machine
(i.e. with a `file:///...` URL) will work perfectly well.

## Writing and reading

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

### The importance of continuations

It is easy to forget that the continuation-passing-style model is
required. For example, you might be tempted to write the above code
as:

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

### Avoiding side effects

Equally, it's important to avoid doing things within a transaction
which have *side-effects*. For example, if you rewrote this code as:

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
