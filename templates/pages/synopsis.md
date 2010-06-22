Intro
=====

Synopsis
--------

Let's suppose we have a module called '`the-good-things`', with some fruit constructors
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
                callback(new(exports.PeeledBanana));
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
                'results in a `PeeledBanana`': function (result) {
                    assert.instanceOf (result, PeeledBanana); 
                }
            }
        }
    }).export(module);

And run them:

    $ vows the-good-things-test.js

Installing
----------

The easiest way to install Vows, is via [npm](http://github.com/isaacs/npm), the node package manager, as so:

    $ npm install vows

This will get you the latest stable version. If you want the bleeding edge, try:

    $ npm install vows@latest

