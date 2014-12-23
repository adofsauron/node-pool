[![Build Status](https://travis-ci.org/coopernurse/node-pool.svg?branch=v3)](https://travis-ci.org/coopernurse/node-pool)

# About

  Generic resource pool.  Can be used to reuse or throttle expensive resources such as
  database connections.
  
## 3.0 Release Warning

Breaking changes:

- Removed `borrow` and `returnToPool` functions, these had been deprecated in v2.
- New resource requests without priority flag now to go back of the queue instead of frint. TODO #74
- Min pool size can now be same as max pool size, instead of defaulting to max -1. TODO #79
- `destroy` now calls `pool.release` and `pool.dispense` itself instead of requiring the user
  to call `pool.release` before `destroy`. TODO #80 #30

## Installation

    $ npm install generic-pool

## Example

### Step 1 - Create pool using a factory object

```js
// Create a MySQL connection pool with
// a max of 10 connections, a min of 2, and a 30 second max idle time
var poolModule = require('generic-pool');
var pool = poolModule.Pool({
    name     : 'mysql',
    create   : function(callback) {
        var Client = require('mysql').Client;
        var c = new Client();
        c.user     = 'scott';
        c.password = 'tiger';
        c.database = 'mydb';
        c.connect();
        
        // parameter order: err, resource
        // new in 1.0.6
        callback(null, c);
    },
    destroy  : function(client) { client.end(); },
    max      : 10,
    // optional. if you set this, make sure to drain() (see step 3)
    min      : 2, 
    // specifies how long a resource can stay idle in pool before being removed
    idleTimeoutMillis : 30000,
     // if true, logs via console.log - can also be a function
    log : true 
});
```

### Step 2 - Use pool in your code to acquire/release resources

```js
// acquire connection - callback function is called
// once a resource becomes available
pool.acquire(function(err, client) {
    if (err) {
        // handle error - this is generally the err from your
        // factory.create function  
    }
    else {
        client.query("select * from foo", [], function() {
            // return object back to pool
            pool.release(client);
        });
    }
});
```

### Step 3 - Drain pool during shutdown (optional)
    
If you are shutting down a long-lived process, you may notice
that node fails to exit for 30 seconds or so.  This is a side
effect of the idleTimeoutMillis behavior -- the pool has a 
setTimeout() call registered that is in the event loop queue, so
node won't terminate until all resources have timed out, and the pool
stops trying to manage them.  

This behavior will be more problematic when you set factory.min > 0,
as the pool will never become empty, and the setTimeout calls will
never end.  

In these cases, use the pool.drain() function.  This sets the pool
into a "draining" state which will gracefully wait until all 
idle resources have timed out.  For example, you can call:

```js
// Only call this once in your application -- at the point you want
// to shutdown and stop using this pool.
pool.drain(function() {
    pool.destroyAllNow();
});
```
    
If you do this, your node process will exit gracefully.
    
    
## Documentation

    Pool() accepts an object with these slots:

                  name : name of pool (string, optional)
                create : function that returns a new resource
                           should call callback() with the created resource
               destroy : function that accepts a resource and destroys it
                   max : maximum number of resources to create at any given time
                         optional (default=1)
                   min : minimum number of resources to keep in pool at any given time
                         if this is set > max, the pool will silently set the min
                         to factory.max - 1
                         optional (default=0)
           refreshIdle : boolean that specifies whether idle resources at or below the min threshold
                         should be destroyed/re-created.  optional (default=true)
     idleTimeoutMillis : max milliseconds a resource can go unused before it should be destroyed
                         (default 30000)
    reapIntervalMillis : frequency to check for idle resources (default 1000),
          returnToHead : boolean, if true the most recently released resources will be the first to be allocated.
                         This in effect turns the pool's behaviour from a queue into a stack. optional (default false)
         priorityRange : int between 1 and x - if set, borrowers can specify their
                         relative priority in the queue if no resources are available.
                         see example.  (default 1)
              validate : function that accepts a pooled resource and returns true if the resource
                         is OK to use, or false if the object is invalid.  Invalid objects will be destroyed.
                         This function is called in acquire() before returning a resource from the pool. 
                         Optional.  Default function always returns true.
                   log : true/false or function -
                           If a log is a function, it will be called with two parameters:
                                                    - log string
                                                    - log level ('verbose', 'info', 'warn', 'error')
                           Else if log is true, verbose log info will be sent to console.log()
                           Else internal log messages be ignored (this is the default)

## Priority Queueing

The pool now supports optional priority queueing.  This becomes relevant when no resources 
are available and the caller has to wait. `acquire()` accepts an optional priority int which 
specifies the caller's relative position in the queue.

```js
 // create pool with priorityRange of 3
 // borrowers can specify a priority 0 to 2
 var pool = poolModule.Pool({
     name     : 'mysql',
     create   : function(callback) {
         // do something
     },
     destroy  : function(client) { 
         // cleanup.  omitted for this example
     },
     max      : 10,
     idleTimeoutMillis : 30000,
     priorityRange : 3
 });

 // acquire connection - no priority - will go at front of line (same as high priority)
 pool.acquire(function(err, client) {
     pool.release(client);
 });

 // acquire connection - high priority - will go into front slot
 pool.acquire(function(err, client) {
     pool.release(client);
 }, 0);

 // acquire connection - medium priority - will go into middle slot
 pool.acquire(function(err, client) {
     pool.release(client);
 }, 1);

 // etc..
```

## Draining

If you know would like to terminate all the resources in your pool before
their timeouts have been reached, you can use `destroyAllNow()` in conjunction
with `drain()`:

```js
pool.drain(function() {
    pool.destroyAllNow();
});
```

One side-effect of calling `drain()` is that subsequent calls to `acquire()`
will throw an Error.

## Pooled function decoration

To transparently handle object acquisition for a function, 
one can use `pooled()`:

```js
var privateFn, publicFn;
publicFn = pool.pooled(privateFn = function(client, arg, cb) {
    // Do something with the client and arg. Client is auto-released when cb is called
    cb(null, arg);
});
```

Keeping both private and public versions of each function allows for pooled 
functions to call other pooled functions with the same member. This is a handy
pattern for database transactions:

```js
var privateTop, privateBottom, publicTop, publicBottom;
publicBottom = pool.pooled(privateBottom = function(client, arg, cb) {
    //Use client, assumed auto-release 
});

publicTop = pool.pooled(privateTop = function(client, cb) {
    // e.g., open a database transaction
    privateBottom(client, "arg", function(err, retVal) {
        if(err) { return cb(err); }
        // e.g., close a transaction
        cb();
    });
});
```

## Pool info

The following functions will let you get information about the pool:

```js
// returns factory.name for this pool
pool.getName()

// returns number of resources in the pool regardless of
// whether they are free or in use
pool.getPoolSize()

// returns number of unused resources in the pool
pool.availableObjectsCount()

// returns number of callers waiting to acquire a resource
pool.waitingClientsCount()

// returns number of maxixmum number of resources allowed by ppol
pool.getMaxPoolSize()
```

## Run Tests

    $ npm install expresso
    $ expresso -I lib test/*.js

## License 

(The MIT License)

Copyright (c) 2010-2014 James Cooper &lt;james@bitmechanic.com&gt;

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
