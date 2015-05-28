*Editor's note: this article originally appeared on Nick Fitzgerald's blog as
[Destructuring Assignment in ES6](http://fitzgeraldnick.com/weblog/50/).*

## What is destructuring assignment?

Destructuring assignment allows you to assign the properties of an array or
object to variables using syntax that looks similar to array or object
literals. This syntax can be extremely terse, while still exhibiting more
clarity than the traditional property access.

Without destructuring assignment, you might access the first three items in an
array like this:

    var first = someArray[0];
    var second = someArray[1];
    var third = someArray[2];

With destructuring assignment, the equivalent code becomes more concise and
read-able:

    var [first, second, third] = someArray;

SpiderMonkey (Firefox's JavaScript engine) already has support for most of
destructuring, but not all of
it. [Track SpiderMonkey's destructuring (and general ES6) support in this bugzilla ticket][es6-bug].

## Destructuring Arrays and Iterables

We already saw one example of destructuring assignment on an array above. The
general form of the syntax is:

    [ variable1, variable2, ..., variableN ] = array;

This will just assign variable1 through variableN to the corresponding item in
the array. If you want to declare your variables at the same time, you can add a
`var`, `let`, or `const` in front of the assignment:

    var [ variable1, variable2, ..., variableN ] = array;
    let [ variable1, variable2, ..., variableN ] = array;
    const [ variable1, variable2, ..., variableN ] = array;

Although `variable` is a misnomer since you can nest patterns as deep as you
would like:

    var [foo, [[bar], baz]] = [1, [[2], 3]];
    console.log(foo);
    // 1
    console.log(bar);
    // 2
    console.log(baz);
    // 3

Furthermore, you can skip over items in the array being destructured:

    var [,,third] = ["foo", "bar", "baz"];
    console.log(third);
    // "baz"

And you can capture all trailing items in an array with a "rest" pattern:

    var [head, ...tail] = [1, 2, 3, 4];
    console.log(tail);
    // [2, 3, 4]

When you access items in the array that are out of bounds or don't exist, you
get the same result you would by indexing: `undefined`.

    console.log([][0]);
    // undefined

    var [missing] = [];
    console.log(missing);
    // undefined

Not that destructuring assignment with an array assignment pattern also works
for any iterable:

    function* fibs() {
      var a = 0;
      var b = 1;
      while (true) {
        yield a;
        [a, b] = [b, a + b];
      }
    }

    var [first, second, third, fourth, fifth, sixth] = fibs();
    console.log(sixth);
    // 5

## Destructuring Objects

Destructuring on objects lets you bind variables to different properties of an
object. You specify the property being bound, followed by the variable you are
binding its value to.

    var robotA = { name: "Bender" };
    var robotB = { name: "Flexo" };

    var { name: nameA } = robotA;
    var { name: nameB } = robotB;

    console.log(nameA);
    // "Bender"
    console.log(nameB);
    // "Flexo"

There is a helpful syntactical shortcut for when the property and variable names
are the same:

    var { foo, bar } = { foo: "lorem", bar: "ipsum" };
    console.log(foo);
    // "lorem"
    console.log(bar);
    // "ipsum"

And just like destructuring on arrays, you can nest and combine further
destructuring:

    var complicatedObj = {
      arrayProp: [
        "Zapp",
        { second: "Brannigan" }
      ]
    };

    var { arrayProp: [first, { second }] } = complicatedObj;

    console.log(first);
    // "Zapp"
    console.log(second);
    // "Brannigan"

When you destructure on properties that are not defined, you get `undefined`:

    var { missing } = {};
    console.log(missing);
    // undefined

One potential gotcha you should be aware of is when you are using destructuring
on an object to assign variables, but not to declare them (when there is no
`let`, `const`, or `var`):

    { blowUp } = { blowUp: 10 };
    // Syntax error

This happens because the JavaScript grammar tells the engine to parse as a block
statement (for example, `{ console }` is a valid block statement). The solution
is to either wrap the pattern or the whole expression in parenthesis:

    ({ safe }) = {};
    ({ andSound } = {});
    // No errors

## Destructuring Values that are not an Object, Array, or Iterable

When you try to use destructuring on `null` or `undefined`, you get a type
error:

    var [blowUp] = null;
    // TypeError: null has no properties

However, you can destructure on other primitive types such as booleans, numbers,
and strings, and get `undefined`:

    var [wtf] = NaN;
    console.log(wtf);
    // undefined

This may come unexpected, but upon further examination it turns out to be
simple. When using an object assignment pattern, the value being destructured is
[required to be coercible to an `Object`][require-object-coercible]. Most types
can be converted to an object, but `null` and `undefined` may not be. When using
an array assignment pattern, the value must [have an iterator][get-iterator].

## Default Values

You can also provide default values for when the property you are destructuring
is not defined:

    var [missing = true] = [];
    console.log(missing);
    // true

    var { x = 3 } = {};
    console.log(x);
    // 3

*Note that SpiderMonkey doesn't implement default values for object assignment
patterns yet. See bug TODO. Default values in array assignment patterns are
supported.*

## Practical Applications of Destructuring

### Function Parameter Definitions

As developers, we can often expose more ergonomic APIs by accepting a single
object with multiple properties as a parameter instead of forcing our API
consumers to remember the order of many individual parameters. We can use
destructuring to avoid repeating this single parameter object whenever we want
to reference one of its properties:

    function removeBreakpoint({ url, line, column }) {
      // ...
    }

This is a simplified snippet of real world code from the Firefox DevTool's
JavaScript debugger (which is also implemented in JavaScript &mdash; yo dog). We
have found this pattern particularly pleasing.

### Configuration Object Parameters

Expanding on the previous example, we can also give default values to the
properties of the objects we are destructuring. This is particularly helpful
when we have an object that is meant to provide configuration and many of the
object's properties already have sensible defaults. For example, jQuery's `ajax`
function takes a configuration object as its second parameter, and could be
re-written like this:

    jQuery.ajax = function (url, {
      async = true,
      beforeSend = noop,
      cache = true,
      complete = noop,
      crossDomain = false,
      global = true,
      // ... more config
    }) {
      // ... do stuff
    };

This avoids repeating `var foo = config.foo || theDefaultFoo;` for each property of
the configuration object.

*Note that default values for object assignment patterns in function parameters
aren't implemented in SpiderMonkey yet. See bug TODO. Default values for array
assignment patterns in function parameters are supported.*

### With the ES6 Iteration Protocol

[ECMAScript 6 also defines an iteration protocol, which we talked about earlier in this series][es6-iter]. When
you iterate over [`Map`s (an ES6 addition to the standard library)][maps], you
get a series of `[key, value]` pairs. We can destructure this pair to get easy
access to both the key and the value:

    var map = new Map();
    map.set(window, "the global");
    map.set(document, "the document");

    for (var [key, value] of map) {
      console.log(key + " is " + value);
    }
    // "[object Window] is the global"
    // "[object HTMLDocument] is the document"

Iterate over only the keys:

    for (var [key] of map) {
      // ...
    }

Or iterate over only the values:

    for (var [,value] of map) {
      // ...
    }

### Multiple Return Values

Although multiple return values aren't baked into the language proper, they
don't need to be because you can return an array and destructure the result:

    function returnMultipleValues() {
      return [1, 2];
    }
    var [foo, bar] = returnMultipleValues();

Alternatively, you can use an object as the container and name the returned
values:

    function returnMultipleValues() {
      return {
        foo: 1,
        bar: 2
      };
    }
    var { foo, bar } = returnMultipleValues();

Both of these patterns end up much better than holding onto the temporary
container:

    function returnMultipleValues() {
      return {
        foo: 1,
        bar: 2
      };
    }
    var temp = returnMultipleValues();
    var foo = temp.foo;
    var bar = temp.bar;

Or using continuation passing style:

    function returnMultipleValues(k) {
      k(1, 2);
    }
    returnMultipleValues((foo, bar) => ...);

### Importing the Subset of a CommonJS Module

Not using ES6 modules yet? Still using CommonJS modules? No problem! When
importing some CommonJS module X, it is fairly common that module X exports more
functions than you actually intend on using. With destructuring, you can be
explicit about which parts of a given module you'd like to use and avoid
cluttering your namespace:

    const { SourceMapConsumer, SourceNode } = require("source-map");














[es6-iter]: https://hacks.mozilla.org/2015/04/es6-in-depth-iterators-and-the-for-of-loop/

[get-iterator]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-getiterator

[require-object-coercible]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-requireobjectcoercible

[maps]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map

[es6-bug]: https://bugzilla.mozilla.org/show_bug.cgi?id=694100
