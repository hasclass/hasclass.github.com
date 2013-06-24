---
layout: hasclass
title: Avoid intermediary objects and Function.apply
---

# JS the fast parts

This is part of a mini-series [Javascript the Fast parts](/articles/javascript-the-fast-parts.html).

## Avoiding Function.apply

Using [Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) is upto 4 times slower than calling the function directly ([jsperf.com/avoiding-apply-or-call](http://jsperf.com/avoiding-apply-or-call)). Clearly, if you're writing a library you should try to avoid `apply` whenever possible. Using `call` you still get a performance hit, but not as severe.

## Example: underscore

Underscore provides a set of functions to maniuplate objects. It primarly provides the method in a functional style (`_.last([1,2,3], 2)` but also adds an object-oriented layer on top of it `_([1,2,3]).last(2)`.


```javascript
// functional
_.last([1,2,3], 2) // => [2,3]
// object-orientated
_([1,2,3])         // => {__wrapped__: [1,2,3]}
_([1,2,3]).last(2)
```

The wrapper objects `last` method calls the corresponding functional method `_.last` with its `__wrapped__` value as the first argument:

```javascript
// Simplified implementation
function _(obj) { this.__wrapped__ = obj; }

_.last = function (obj, len) {
  // implementation
};

// Call the functional method to stay DRY
_.prototype.last = function(len) {
  return _.last(this.__wrapped__, len);
}
```

In order to not repeat yourself and increase file size of the library, the OO-methods are created dynamically.

### Refactor into a generic method

Instead of adding a hard-coded method body for all the methods they are added dynamically. If we refactor the `_.prototype.last` into a generic method that works with any amount of arguments we get something like this.

```javascript
// simplified implementation
function wrapFunction (func) {
  return function () {
    var args = [this.__wrapped__];
    Array.prototype.push.apply(args, arguments);
    return func.apply(context, args);
  }
}

Wrapper.prototype.last = wrapFunction(_.last);
```

The wrapFunction returns a function that prepends `this` to the method's arguments, calls the actual method and returns its return value. The following code shows how that would look like if it was hard-coded:

```javascript
// Generates a method like this:
Wrapper.prototype.last = function () {
 var args = [this.__wrapped__];
 Array.prototype.push.apply(args, arguments);
 return _.last.apply(_, args);
}
```

Unfortunately wrapFunction creates a big overhead. Everytime we call `_(obj).last()` the wrapped function does the following:

* Create an intermediate extra array `args`
* Pushes arguments to that array using `apply`
* `args` are `apply`ed to the actual function

While `apply` is slow by itself, it also almost always requires the creation and manipulation of intermediary arrays or "array-likes".

### Avoiding apply and intermediate objects

All of this is unnecessary when you pass no arguments. So a trivial optimization would be to check for this case:

```javascript
// optimize for zero args
function wrapFunction (func) {
  return function () {
    var len = arguments.length;
    var val = this.__wrapped;
    // Return early for commong case
    if (len === 0) return func(val);

    // slow fallback for arguments
    var args = [this.__wrapped__];
    Array.prototype.push.apply(args, arguments);
    return func.apply(_, args);
  }
}
```

The same strategy can be applied to x arguments.

```javascript
function wrapFunction (func) {
  return function () {
    var val = this.__wrapped__;
    switch (arguments.length) {
      case 0: return func(val);
      case 1: return func(val, arguments[0]);
      case 2: return func(val, arguments[0], arguments[1]);
      // repeat x times..
    }
    // slow fallback for more then x arguments
    var args = [val];
    Array.prototype.push.apply(args, arguments);
    return func.apply(_, args);
  }
}
```

That makes wrapFunction performs 3-10 times faster depending on your JS engine.

### Optimized apply

If that's not enough by now, let's extract above and make a faster Function.prototype.apply routine.

```javascript
function fastApply (fn, context, args) {
  switch(args.length) {
    case 0: return fn.call(context);
    case 1: return fn.call(context, args[0]);
    case 2: return fn.call(context, args[0], args[1]);
  }
  return fn.apply(context, args);
}

function noop(a,b) {};

fastApply(noop, {}, [1,2]);

noop.apply({}, [1,2]);
```

And again, just by avoiding `apply` we get a significant boost; [2-10 x faster](http://jsperf.com/custom-apply). There's various variants on above functions. If you don't want to apply an actual context and just want to call a method with arguments in an array:

```javascript
function fastApplyNoContext (fn, args) {
  switch(args.length) {
    case 0: return fn();
    case 1: return fn(args[0]);
  }
  return fn.apply(null, args);
}
```

Or simply extract the `wrapFunction` into it's own function.

```javascript
function fastApplyPrepend (fn, obj, args) {
  switch(args.length) {
    case 0: return fn(obj)
    case 1: return fn(obj, args[0])
  }
  // ...
}
```

## Final Words

This article focused a lot on avoiding `apply` which can give you a performance boost of a factor of 2-10 across browsers, new and old, Chrome, Safari or Firefox. But `apply` was not just the only bad smell in the code. By avoiding apply we also avoided the steps of creating an extra array and pushing elements of the function arguments into it.







