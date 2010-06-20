Guide
=====

How topics work
---------------

Understanding *topics* is the key to understanding how Vows works. Unlike other testing frameworks,
Vows forces a clear separation between the element which is tested, the *topic*, and the actual tests, the *vows*.

Let's start with a simple example:

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

Same thing. Topics can be functions too. Now what if we have multiple vows?

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

Structure of a test suite
-------------------------

### Putting it together #

Test suites in Vows are the largest unit of tests. The convention is to have one test suite
per file, and have the suite's subject match the file name. Test suites are created with `vows.describe`.

    var suite = vows.describe('subject');

Tests are added to suites in *batches*. This is done with the `addBatch` method.

    suite.addBatch({});

You can add as many batches to a suite as you want. Batches are executed *sequentially*.

    suite.addBatch({/* 1st */}).addBatch({/* 2nd */}).addBatch({/* 3rd */});

Batches are useful when you want to test functionality in a certain order.

### Running the suite #

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

*subject-test.js*

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

*subject-test.js*

    // A test suite, describing 'subject'

    vows.describe('subject') // Create the suite, describing 'subject'
        .addBatch({})         // Add the 1st batch
        .addBatch({})         // Add a 2nd batch
        .addBatch({})         // Add a 3rd batch
        .export(module);     // Export it

Structure of a batch
--------------------

» A *batch* is an object literal, representing a structure of nested *contexts*.

» A *context* is an object with a *topic*, zero or more *vows* and zero or more *sub-contexts*.

» A *vow* is a function which receives the *topic* as an argument, and runs assertions on it. 

With that in mind, we can imagine a structure like this:

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
    }

Here's an example:

    {                                                      // Batch
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
    }




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

When this function is *called*, it passes on the arguments it received to the test functions,
one by one, as if the values were returned by the topic function itself.

In essence, this allows us to decouple the callback from the async function call. It's equivalent to:

    fs.stat('~/FILE', function (err, stat) {
        assert.isNull    (err,       'can be accessed');
        assert.isObject  (stat,      'can be accessed');
        assert.isNotZero (stat.size, 'is not empty');
    });

Except it allows Vows to keep track of all the asynchronous calls, and warn you if something
hasn't returned.


