## Introduction ##

This file provides a framework for writing testcases. It is intended to
provide a convenient API for making common assertions, and to work both
for testing synchronous and asynchronous DOM features in a way that
promotes clear, robust, tests.

## Basic Usage ##

To use this file, import the script and the testharnessreport script into
the test document:

``` html
<script src="/resources/testharness.js"></script>
<script src="/resources/testharnessreport.js"></script>
```

Within each file one may define one or more tests. Each test is atomic in the sense that a single test has a single result (pass/fail/timeout). Within each test one may have a number of asserts. The test fails at the first failing assert, and the remainder of the test is (typically) not run.

If the file containing the tests is a HTML file with an element of id "log" this will be populated with a table containing the test results after all the tests have run.

NOTE: By default tests must be created before the load event fires. For ways to create tests after the load event, see "Determining when all tests are complete", below.

## Synchronous Tests ##

To create a synchronous test use the test() function:

``` js
test(test_function, name, properties)
```

test_function is a function that contains the code to test. For example a trivial passing test would be:

```js
test(function() {
  assert_true(true);
}, "assert_true with true");
```

The function passed in is run in the `test()` call.

`properties` is an object that overrides default test properties. The recognised properties are:

* `timeout` - the test timeout in ms.

e.g.

```js
test(test_function, "Sample test", { timeout:1000 });
```

would run `test_function` with a timeout of 1s.

## Asynchronous Tests ##

Testing asynchronous features is somewhat more complex since the result of a test may depend on one or more events or other callbacks. The API provided for testing these features is indended to be rather low-level but hopefully applicable to many situations.

To create a test, one starts by getting a Test object using async_test:

```js
async_test(name, properties);
```

e.g.

```js
var t = async_test("Simple async test");
```

Assertions can be added to the test by calling the step method of the test object with a function containing the test assertions:

```js
t.step(function() { assert_true(true); });
```

When all the steps are complete, the `done()` method must be called:

```js
t.done();
```

The properties argument is identical to that for `test()`.

In many cases it is convenient to run a step in response to an event or a callback. A convenient method of doing this is through the `step_func` method which returns a function that, when called runs a test step. For example 

```js
object.some_event = t.step_func(function(e) {
  assert_true(e.a);
});
```

## Making assertions ##

Functions for making assertions start with `assert_`. The best way to get a list is to look in this file for functions names matching that pattern. The general signature is

```js
assert_something(actual, expected, description);
```

although not all assertions precisely match this pattern e.g. `assert_true` only takes actual and description as arguments.

The description parameter is used to present more useful error messages when a test fails

NOTE: All asserts must be located in a `test()` or a step of an `async_test()`. asserts outside these places won't be detected correctly by the harness and may cause a file to stop testing.

## Setup ##

Sometimes tests require non-trivial setup that may fail. For this purpose there is a `setup()` function, that may be called with one or two arguments.
The two argument version is:

```js
setup(func, properties);
```

The one argument versions may omit either argument. `func` is a function to be run synchronously. `setup()` becomes a no-op once
any tests have returned results. Properties are global properties of the test harness. Currently recognised properties are:

* `timeout` - The time in ms after which the harness should stop waiting for tests to complete (this is different to the per-test timeout because async tests do not start their timer until .step is called)
* `explicit_done` - Wait for an explicit call to `done()` before declaring all tests complete (see below).
* `output_document` - The document to which results should be logged. By default this is the current document but could be an ancestor document in some cases e.g. a SVG test loaded in an HTML wrapper

## Determining when all tests are complete ##

By default the test harness will assume there are no more results to come
when:

1. There are no Test objects that have been created but not completed
2. The load event on the document has fired

This behaviour can be overridden by setting the `explicit_done` property to `true` in a call to `setup()`. If `explicit_done` is `true`, the test harness will not assume it is done until the global `done()` function is called. Once `done()` is called, the two conditions above apply like normal.

## Generating tests ##

NOTE: this functionality may be removed

There are scenarios in which is is desirable to create a large number of (synchronous) tests that are internally similar but vary in the parameters used. To make this easier, the generate_tests function allows a single function to be called with each set of parameters in a list:

```js
generate_tests(test_function, parameter_lists);
```

For example:

```js
generate_tests(assert_equals, [
    ["Sum one and one", 1+1, 2],
    ["Sum one and zero", 1+0, 1]
  ]);
```

Is equivalent to:

```js
test(function() { assert_equals(1+1, 2); }, "Sum one and one");
test(function() { assert_equals(1+0, 1); }, "Sum one and zero");
```

Note that the first item in each parameter list corresponds to the name of the test.

## Callback API ##

The framework provides callbacks corresponding to 3 events:

* `start` - happens when the first Test is created
* `result` - happens when a test result is received
* `complete` - happens when all results are received

The page defining the tests may add callbacks for these events by calling the following methods:


* `add_start_callback(callback)` - callback called with no arguments
* `add_result_callback(callback)` - callback called with a test argument
* `add_completion_callback(callback)` - callback called with an array of tests and an status object

tests have the following properties:

* `status`: A status code. This can be compared to the `PASS`, `FAIL`, `TIMEOUT` and `NOTRUN` properties on the test object
* `message`: A message indicating the reason for failure. In the future this will always be a string

The status object gives the overall status of the harness. It has the following properties:

* `status`: Can be compared to the `OK`, `ERROR` and `TIMEOUT` properties
* `message`: An error message set when the status is `ERROR`

## External API ##

In order to collect the results of multiple pages containing tests, the test harness will, when loaded in a nested browsing context, attempt to call certain functions in each ancestor browsing context:

* `start` - `start_callback`
* `result` - `result_callback`
* `complete` - `completion_callback`

These are given the same arguments as the corresponding internal callbacks described above.

## List of assertions ##

* `assert_true(actual, description)`

    asserts that `actual` is strictly `true`

* `assert_false(actual, description)`

    asserts that `actual` is strictly `false`

* `assert_equals(actual, expected, description)`

    asserts that `actual` is the same value as `expected`

* `assert_not_equals(actual, expected, description)`

    asserts that `actual` is a different value to `expected`. Yes, this means that `expected` is a misnomer

* `assert_in_array(actual, expected, description)`

    asserts that `expected` is an `Array`, and `actual` is equal to one of the members -- `expected.indexOf(actual) != -1`

* `assert_array_equals(actual, expected, description)`

    asserts that `actual` and `expected` have the same length and the value of each indexed property in `actual` is the strictly equal to the corresponding property value in `expected`

* `assert_approx_equals(actual, expected, epsilon, description)`

    asserts that `actual` is a number within +/- `epsilon` of `expected`

* `assert_regexp_match(actual, expected, description)`

    asserts that `actual` matches the regexp `expected`

* `assert_own_property(object, property_name, description)`

    assert that `object` has own property `property_name`

* `assert_inherits(object, property_name, description)`

    assert that `object` does not have an own property named `property_name` but that `property_name` is present in the prototype chain for `object`

* `assert_idl_attribute(object, attribute_name, description)`

    assert that an `object` that is an instance of some interface has the attribute `attribute_name` following the conditions specified by WebIDL.

* `assert_readonly(object, property_name, description)`

    assert that property `property_name` on `object` is readonly.

* `assert_throws(code, func, description)`

    * `code` - a `DOMException`/`RangeException` code as a string, e.g. `"HIERARCHY_REQUEST_ERR"`
    * `func` - a function that should throw

    assert that `func` throws a `DOMException` or `RangeException` (as appropriate) with the given code. If an object is passed for code instead of a string, checks that the thrown exception has a property called `"name"` that matches the property of code called `"name"`. Note, this function will probably be rewritten sometime to make more sense.

* `assert_unreached(description)`

    asserts if called. Used to ensure that some codepath is *not* taken e.g. an event does not fire.

* `assert_exists(object, property_name, description)`

    **deprecated**

    asserts that `object` has an own property `property_name`

* `assert_not_exists(object, property_name, description)`

    **deprecated**

    assert that `object` does not have own property `property_name`
