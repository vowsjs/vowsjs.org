Reference
=========

Test runner
-----------

### Running tests #

    $ vows test-1.js test-2.js
    $ vows tests/*

Watch mode

    $ vows -w
    $ vows --watch

### Options #

- `-v`, `--verbose`: Verbose mode
- `-m'goo'`: String matching: Only run tests with `'goo'` in their title
- `-r'goo$'`: Regexp matching: Only run tests with a title ending with 'goo'

- `--json`: Use JSON reporter
- `--spec`: Use Spec reporter

Reporters
---------

- Dot Matrix: `'dot-matrix'` (default)
- Spec: `'spec'`
- JSON: `'json'`

Assertion functions
-------------------

### equality #

- assert.equal

        assert.equal(4, 4);

- assert.notEqual

        assert.notEqual(4, 2);

- assert.strictEqual

        assert.strictEqual(4 > 2, true);

- assert.strictNotEqual

        assert.strictNotEqual(1, true);

### type #

- assert.isFunction

        assert.isFunction([].slice);
    
- assert.isObject

        assert.isObject({goo:true});

- assert.isString

        assert.isString('goo');

- assert.isArray

        assert.isArray([4, 2]);

- assert.isNumber

        assert.isNumber(42);

- assert.isBoolean

        assert.isBoolean(true);

- assert.isTrue

        assert.isTrue(true);

- assert.isFalse

        assert.isFalse(false);

- assert.isNull

        assert.isNull(null);

- assert.isNotNull

        assert.isNotNull(undefined);

- assert.isNaN

        assert.isNaN(0/0);

- assert.isUndefined

        assert.isUndefined('goo'[9]);

- assert.typeOf

        assert.typeOf(42, 'number');

- assert.instanceOf

        assert.instanceOf('hello', String);

### properties #

- assert.include

        assert.include([4, 2, 0], 2);
        assert.include({goo:true}, 'goo');
        assert.include('goo', 'o');

- assert.match

        assert.match('hello', /^[a-z]+/);

- assert.length

        assert.length([4, 2, 0], 3);
        assert.length('goo', 3);

- assert.isEmpty

        assert.isEmpty([]);
        assert.isEmpty({});
        assert.isEmpty("");

### exceptions #

- assert.throws
- assert.doesNotThrow

The Good Things
---------------

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

