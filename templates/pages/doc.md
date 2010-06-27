Guide
=====

To understand Vows, we're going to start with a general overview of the different components involved in writing tests,
and then go through some of them in more detail.

Structure of a test suite
-------------------------

Test suites in Vows are the largest unit of tests. The convention is to have one test suite
per file, and have the suite's subject match the file name. Test suites are created with `vows.describe`.

    var suite = vows.describe('subject');

Tests are added to suites in *batches*. This is done with the `addBatch` method.

    suite.addBatch({});

You can add as many batches to a suite as you want. Batches are executed *sequentially*.

    suite.addBatch({/* run 1st */}).addBatch({/* 2nd */}).addBatch({/* 3rd */});

Chaining batches is useful when you want to test functionality in a certain order.

Batches contain *contexts*, which describe different components and states you want to test.

    suite.addBatch({
       'A context': {},
       'Another context': {}
    });

Contexts are executed *in parallel*, they are fully asynchronous. The order in which they finish is therefore undefined.

Contexts usually contain *topics* and *vows*, which in combination define your tests.

    suite.addBatch({
       'A context': {
            topic: function () {/* Do something async */},
            'I am a vow': function (topic) {
                /* Test the result of the topic */
            }
        },
       'Another context': {}
    });

Contexts can contain *sub-contexts* which get executed as soon as the parent context finishes:

    suite.addBatch({
       'A context': {
            topic: function () {/* Do something async */},
            'I am a vow': function (topic) {
                /* Test the result of the topic */
            }
            'A sub-context': {/* Executed when the tests above finish running */}   
        },
       'Another context': {/* Executed in parallel to 'A context' */}
    });


So to recap:

» A *Suite* is an object which contains zero or more *batches*, and can be executed or exported.

» A *batch* is an object literal, representing a structure of nested *contexts*.

» A *context* is an object with an optional *topic*, zero or more *vows* and zero or more *sub-contexts*.

» A *topic* is either a value or a function which can execute asynchronous code.

» A *vow* is a function which receives the *topic* as an argument, and runs assertions on it.

With that in mind, we can imagine a structure like this:

    Suite {
        Batch {
            Context {
                Topic,
                Context {
                    Topic, Vow, Vow, Vow,
                    Context {
                        Topic, Vow
                    }
                }
            }
        },
        Batch {}
    }

Here's an annotated example:

    vows.describe('Array').addBatch({                      // Batch
        'An array': {                                      // Context
            'with 3 elements': {                           // Sub-Context
                topic: [1, 2, 3],                          // Topic

                'has a length of 3': function (topic) {    // Vow
                    assert.equal(topic.length, 3);
                }
            },
            'with zero elements': {                        // Sub-Context
                topic: [],                                 // Topic

                'has a length of 0': function (topic) {    // Vow
                    assert.equal(topic.length, 0);
                },
                'returns *undefined*, when `pop()`ed': function (topic) {
                    assert.isUndefined(topic.pop());
                }
            }
        }
    });

How topics work
---------------

Understanding *topics* is one of the keys to understanding Vows. Unlike other testing frameworks,
Vows forces a clear separation between the element which is tested, the *topic*, and the actual tests, the *vows*.

Let's start with a simple example of a context:

    { topic: 42,
      'should be equal to 42': function (topic) {
        assert.equal (topic, 42);
      }
    }

So this shows us that the value of the topic is passed down to our test function (refered to as a *vow* from now on) as an argument.
Simple enough. Now let's look at an equivalent example, written differently:

    { topic: function () { return 42 },
      'should be equal to 42': function (topic) {
        assert.equal (topic, 42);
      }
    }

Same thing. Topics can be functions too. The return value becomes the topic. Now what if we have multiple vows?

    { topic: function () { return 42 },
      'should be a number': function (topic) {
        assert.isNumber (topic);
      },
      'should be equal to 42': function (topic) {
        assert.equal (topic, 42);
      }
    }

It works as expected, the value is passed down to each *vow*. Note that the topic function is **only run once**.

### Scope #

Sometimes, you might need a parent topic's value, from inside a child topic. This is easy, because there is
a notion of topic *scope*. Let's look at an example:

    { topic: new(DataStore),
      'should respond to `get()` and `put()`': function (store) {
        assert.isFunction (store.get);
        assert.isFunction (store.put);
      },
      'calling `get(42)`': {
        topic: function (store) { return store.get(42) },
        'should return the object with id 42': function (topic) {
          assert.equal (topic.id, 42);
        }
      }
    }

In the example above, the value of the top-level topic is passed as an argument to the inner topic, in the same manner
it's passed to the vows. For clarity, I named both arguments which refer to the outer topic as `store`.

Note that the scoping isn't limited to a single level. Consider:

    topic: function (a, /* Parent topic                     */
                     b, /* Parent of parent topic           */
                     c  /* Parent of parent of parent topic */) {}

So the parent topics are passed along to each topic function in the certain order: the immediate parent is always the first
argument (`a`), and the outer topics follow (`b`, then `c`), like the layers of an onion.

Running a suite
---------------

The simplest way to run a test suite, is with the `run` method:

    vows.describe('subject').addBatch({/* ... */}).run();

The `run` method takes an optional callback, which is called when all tests are done running.
The test results are passed to the callback (if provided), as an object:

    { honored: 145,
      broken:    4,
      errored:   1,
      pending:   0,
      total:   150,
      time:  5.491
    };

Now if we want to execute this test suite, assuming it's in *subject-test.js*, we just do:

    $ node subject-test.js

The results will be printed to the console with the default reporter, `'dot-matrix'`.

### Exporting the suite #

When your tests become more complex, spanning multiple files, you're going to need a way to run
them as a single entity.

Vows has a test runner called `vows`, which you can use to run multiple test suites at once.
To make use of it, you need to export your tests, instead of just running them. There's a couple
of ways to do that, the easiest is through the `export` method:

    // subject-test.js

    vows.describe('subject').addBatch({/* ... */}).export(module);

`export` takes one argument, the module you want to export the test suite to. Fortunately,
node provides a global variable called `module`, which is a reference to the current module.

Now to run that file with the test runner, we can do:

    $ vows subject-test.js

The result should be identical to running it directly with `node`. The difference is that we can now do:

    $ vows test/*

to run all the tests in our *test/* folder, and get combined results. We can also pass options to `vows`.
For example, to get a "spec style" output, pass the `--spec` flag. The reference section has more information on
the different options you can pass to it.

Another way to export your test suites is by simply adding them to the `exports` object, the same way you would export
an API to a library:

    exports.suite1 = vows.describe('suite one');
    exports.suite2 = vows.describe('suite two');

### So let's recap #

    // subject-test.js
    // A test suite, describing 'subject'

    vows.describe('subject') // Create the suite, describing 'subject'
        .addBatch({})        // Add the 1st batch
        .addBatch({})        // Add a 2nd batch
        .addBatch({})        // Add a 3rd batch
        .export(module);     // Export it

Writing asynchronous tests
--------------------------

> Before diving into asynchronous testing, make sure you read the section about *topics*.

Let's say we want to test that a certain file exists, and satisfies a couple criteria.

As you know, we don't 'return' a value from an asynchronous function call--the value is
passed to the callback function. So how can we do that with *topics*? Take a look:

    { topic: function () {
        fs.stat('~/FILE', this.callback);
      },
      'can be accessed': function (err, stat) {
        assert.isNull   (err);        // We have no error
        assert.isObject (stat);       // We have a stat object
      },
      'is not empty': function (err, stat) {
        assert.isNotZero (stat.size); // The file size is > 0
      }
    }

The key here is the special '`this.callback`' function, which is available inside all topics.

When `this.callback` is *called*, it passes on the arguments it received to the test functions,
one by one, as if the values were returned by the topic function itself.

In essence, this allows us to decouple the callback from the async function call.

This is how Vows keeps track of all the asynchronous calls, and can warn you if something
hasn't returned.

> Note that topics which make use of '`this.callback`' must not return anything. And likewise, topics
which do not return anything must make use of '`this.callback`'.

### Promises #

Vows also supports promise-based async out of the box, so if that works better for your purpose,
you can return an instance of `EventEmitter` from a topic, and the tests will be run when it
emits a `"success"` or `"error"` event:

    { topic: function () {
        var promise = new(events.EventEmitter);

        fs.stat('~/FILE', function (e, res) {
            if (e) { promise.emit('error', e) }
            else   { promise.emit('success', res) }
        });
        return promise;
      },
      'can be accessed': function (err, stat) {
        assert.isNull   (err);        // We have no error
        assert.isObject (stat);       // We have a stat object
      },
      'is not empty': function (err, stat) {
        assert.isNotZero (stat.size); // The file size is > 0
      }
    }

### Order of execution and parallelism #

We talked about how batches and contexts are executed briefly,
but it's now time to delve into it in more detail:

    { topic: function () {
        fs.stat('~/FILE', this.callback);
      },
      'after a successful `fs.stat`': {
        topic: function (stat) {
          fs.open('~/FILE', "r", stat.mode, this.callback);
        },
        'after a successful `fs.open`': {
          topic: function (fd, stat) {
            fs.read(fd, stat.size, 0, "utf8", this.callback);
          },
          'we can `fs.read` to get the file contents': function (data) {
            assert.isString (data);
          }
        }
      }
    }

In the example above, we make use of nested contexts to mimic nested callbacks. As you can tell,
the result of the parent topic is passed down to its children, as arguments.

This example as a whole is therefore mostly sequential, while remaining asynchronous.

---

Now let's look at an example which uses parallel tests to check for some devices:

    { '/dev/stdout': {
        topic:    function () { path.exists('/dev/stdout', this.callback) },
        'exists': function (result) { assert.isTrue(result) }
      },
      '/dev/tty': {
        topic:    function () { path.exists('/dev/tty', this.callback) },
        'exists': function (result) { assert.isTrue(result) }
      },
      '/dev/null': {
        topic:    function () { path.exists('/dev/null', this.callback) },
        'exists': function (result) { assert.isTrue(result) }
      }
    }

So in this case, the tests can finish in any order, and must not rely on each other. The test
suite will exit when the last I/O call completes, and the assertions for it are run.

In other words, *sibling contexts* are executed in parallel, while *nested contexts* are
executed sequentially. Note that this all happens asynchronously, so while some contexts
may be waiting for a parent context to finish, sibling contexts can still execute in the meantime.

Assertions
----------

Vows extends the assertion module which comes bundled with node, with many useful functions,
as well as better error reporting for the existing ones.

It's always best to use the more specific assertion functions when testing a value,
you'll get much better error reporting, because your intention is clearer.

Let's say we have the following array:

    var ary = [1, 2, 3];

and try to assert that it has 5 elements. With the built-in `assert.equal`,
we would do something like this:

    assert.equal(ary.length, 5);

And get the following error:

    expected 5, got 3

Now let's try that with one of our more specific assertion functions, `assert.length`:

    assert.length(ary, 5);

This reports the following error:

    expected [1, 2, 3] to have 5 elements

Other useful assertion functions bundled with vows include `assert.match`, `assert.instanceOf`,
`assert.include` and `assert.isEmpty`--head over to the [reference](/reference#assert) to get the full list.

Macros
------

Sometimes, it's useful to abstract tests which are used throughout the test suite. A *batch* in Vows,
is a tree-like data structure--an Object literal to be precise. This proves to be pretty powerful, as you'll see.

One of the things I have to test in the  majority of the code I write are HTTP status codes. So let's first look
at the straightforward way of doing this, given an asynchronous `client` library:

    { topic: function () {
        client.get('/resources/42', this.callback);
      },
      'should respond with a 200 OK': function (e, res) {
        assert.equal (res.status, 200);
      }
    }

Not too bad. But we might have a hundred of these, if we're testing an API. So let's see what we can do with macros:

    function assertStatus(code) {
        return function (e, res) {
            assert.equal (res.status, code);
        };
    }

This is a function which takes a status code, and returns a function which tests for that status code. We can now
improve our test like this:

    { topic: function () {
        client.get('/resources/42', this.callback);
      },
      'should respond with a 200 OK': assertStatus(200)
    }

Much better. How about the topic? Let's write a macro for our API calls:

    var api = {
        get: function (path) {
            return function () {
                client.get(path, this.callback);
            };
        }
    };

And rewrite our tests:

    { topic: api.get('/resources/42'),
      'should respond with a 200 OK': assertStatus(200)
    }

Fantastic. Here's a an example of what these macros could look like:

    {   'GET /': {
            topic: api.get('/'),
            'shoud respond with a 200 OK': assertStatus(200)
        },
        'POST /': {
            topic: api.post('/'),
            'shoud respond with a 405 Method not allowed': assertStatus(405)
        },
        'GET /resources (no api-key)': {
            topic: api.get('/resources'),
            'shoud respond with a 403 Forbidden': assertStatus(403)
        },
        'GET /resources?apikey=af816e859c249fe'
            topic: api.get('/resources?apikey=af816e859c249fe'),
            'shoud return a 200 OK': assertStatus(200),
            'should return a list of resources': function (res) {
                assert.isArray (res.body);
            }
        }
    }

Can we push it further? Of course we can, and this is when it gets *really* interesting. I'm going to
show you how you can generate contextual tests.

Instead of having a separate function which generates a *topic*, and another one which generates
a *vow*, we're going to have a function which generates a *context* which contains both a topic and a vow.

The topic will perform a *contextual* request. This is the interesting part: we're going to parse
the context description to generate the api requests. So the test will be encoded within its
description. Let's look at a possible implementation:

    //
    // Send a request and check the response status.
    //
    function respondsWith(status) {
        var context = {
            topic: function () {
                // Get the current context's name, such as "POST /"
                // and split it at the space.
                var req    = this.context.name.split(/ +/), // ["POST", "/"]
                    method = name[0].toLowerCase(),         // "post"
                    path   = name[1];                       // "/"

                // Perform the contextual client request,
                // with the above method and path.
                client[method](path, this.callback);
            }
        };
        // Create and assign the vow to the context.
        // The description is generated from the expected status code
        // and status name, from node's http module.
        context['should respond with a ' + status + ' '
               + http.STATUS_CODES[status]] = assertStatus(status);

        return context;
    }

Now the first three contexts of our batch can be re-written as:

    { 'GET  /':                   respondsWith(200),
      'POST /':                   respondsWith(405),
      'GET  /resources (no key)': respondsWith(403)
    }

And when run, we get:

<div class="report"><pre class="report">
GET  /
  ✓ <span class="vow">should respond with a 200 OK</span>
POST /
  ✓ <span class="vow">should respond with a 405 Method Not Allowed</span>
GET  /resources (no key)
  ✓ <span class="vow">shoud respond with a 403 Forbidden</span>
</pre></div>

The fourth context is a little more complex, as it has two vows, but I'll let you figure that
one out!


