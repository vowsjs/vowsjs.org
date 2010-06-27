
Synopsis
========

Vows is a [behavior driven development](http://en.wikipedia.org/wiki/Behavior_Driven_Development)
framework for [Node.js](http://nodejs.org).

Vows was built from the ground up to test asynchronous code. It executes your tests in parallel when it makes sense,
and sequentially when there are dependencies.

Emphasis was put onspeed of execution, clarity and user experience.

Here's a simple example, describing 'division by zero':

    // division-by-zero-test.js

    var vows = require('vows'),
        assert = require('assert');

    vows.describe('Division by Zero').addBatch({
        'when dividing a number by zero': {
            topic: function () { return 42 / 0 },

            'we get Infinity': function (topic) {
                assert.equal (topic, Infinity);
            }
        },
        'but when dividing zero by zero': {
            topic: function () { return 0 / 0 },

            'we get a value which': {
                'is not a number': function (topic) {
                    assert.isNaN (topic);
                },
                'is not equal to itself': function (topic) {
                    assert.notEqual (topic, topic);
                }
            }
        }
    }).run();

And run it:

    $ node division-by-zero-test.js

---

And now, a little more involved example--let's suppose we have a module called '`the-good-things`', with some fruit constructors
in it:

    exports.Strawberry = function () {
        this.color = '#ff0000';
    };
    exports.Strawberry.prototype = {
        isTasty: function () { return true }
    };

    exports.Banana = function () {
        this.color = '#fff333';
    };
    exports.Banana.prototype = {
        peel: function (callback) {
            process.nextTick(function () {
                callback(null, new(exports.PeeledBanana));
            });
        },
        peelSync: function () { return new(exports.PeeledBanana) }
    };

    exports.PeeledBanana = function () {};

Now write some tests in *the-good-things-test.js*:

    var vows = require('vows'),
        assert = require('assert');

    var theGoodThings = require('the-good-things');

    var Strawberry   = theGoodThings.Strawberry,
        Banana       = theGoodThings.Banana,
        PeeledBanana = theGoodThings.PeeledBanana;

    vows.describe('The Good Things').addBatch({
        'A strawberry': {
            topic: new(Strawberry),

            'is red': function (strawberry) {
                assert.equal (strawberry.color, '#ff0000');
            },
            'and tasty': function (strawberry) {
                assert.isTrue (strawberry.isTasty());
            }
        },
        'A banana': {
            topic: new(Banana),

            'when peeled *synchronously*': {
                topic: function (banana) {
                    return banana.peelSync();
                },
                'returns a `PeeledBanana`': function (result) {
                    assert.instanceOf (result, PeeledBanana);
                }
            }
            'when peeled *asynchronously*': {
                topic: function (banana) {
                    banana.peel(this.callback);
                },
                'results in a `PeeledBanana`': function (err, result) {
                    assert.instanceOf (result, PeeledBanana);
                }
            }
        }
    }).export(module);

And run them:

    $ vows the-good-things-test.js

