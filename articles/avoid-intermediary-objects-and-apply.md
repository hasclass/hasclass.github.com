---
layout: hasclass
title: Avoid intermediary objects and Function.apply
---

# JavaScript the Fast Parts

This is part of a mini-series [Javascript the Fast parts](/articles/javascript-the-fast-parts.html).

## Avoiding Function.apply

This article focuses on ways to avoid the use of `Function.prototype.apply`. Many JavaScript libraries depend on `apply` to implement core functionality. The versatility of this function comes with a price, it is [2-10 slower](http://jsperf.com/avoiding-apply-or-call) than calling a function directly. Additionally, `apply` usually needs to create  intermediary objects which is another performance bad smell.


Outline:

* [Explanation of intermediary objects](#intermediary_objects)
* [Look at the `_.wrap` function of underscore](#wrap)
* [Make it perform 2-20 times faster](#optimize_wrap)
* [Extract the functionality for re-use](#extract_for_reuse)
* [Create the applyFast function that outperforms apply by 2-10 times](#applyFast)

<a name="intermediary_objects" /></a>

## Intermediary Objects

All objects that are created within a function and are not part of the returned value I call intermediary objects. They add overhead to a function and should be avoided.

```javascript
function hello(who) {
  var greeting = 'hello';
  var sep      = ' ';
  var args = [greeting, who];
  return args.join(sep);
}
```

In the hello function we have 3 intermediary objects: `greeting`, `args` and `sep`. To be precise `greeting` and `sep` are JavaScript literals and not objects, so they pose no (significant) overhead. The real offender is the `args` array which is a fully fledged object, and object-creation doesn't come for free. This simple function can be refactored easily:

```
function hello(who) {
  var greeting = 'hello ';
  return greeting + who;
}
```

After this contrived example let's look at the real-world.

### Avoiding apply and intermediary objects


<a name="wrap" /></a>

### _.wrap(function, wrapper)

From the docs: Wraps the first function inside of the wrapper function, passing it as the first argument. This allows the wrapper to execute code before and after the function runs, adjust the arguments, and execute it conditionally.

```javascript
var hello = function(name) { return "hello: " + name; };
hello = _.wrap(hello, function(func) {
  return "before, " + func("moe") + ", after";
});
hello();   // => 'before, hello: moe, after'
```

The implementation of _.wrap is short and concise:

```javascript
_.wrap = function(func, wrapper) {
  return function() {
    var args = [func];
    push.apply(args, arguments);
    return wrapper.apply(this, args);
  };
};
```

However there are the following inefficencies:

* Creates an intermediary array `args`
* Pushes arguments to that array. The second `apply` is actually a performant idiom
* `apply` the `args` array to the `wrapper` function

While inefficient, this generic solution works for 0 to n arguments of the wrapper function. But all of this is work is unnecessary for wrapper functions that expect no arguments.

<a name="optimize_wrap" /></a>

### Optimize _.wrap for a common special case

A trivial optimization would be to optimize when arguments is empty. In that case we could simply use the much faster `call` operation.

```javascript
_.wrap = function(func, wrapper) {
  return function() {
    if (arguments.length === 0) return wrapper.call(this, func);
    // fallback
    var args = [func];
    push.apply(args, arguments);
    return wrapper.apply(this, args);
  }
};
```

The new function now is very fast when no arguments are given. And otherwise it falls back to the slower original implementation.

### Optimize _.wrap for all cases

The optimization we did for zero arguments can be repeated for cases where the arguments object is not empty.

```javascript
_.wrap = function(func, wrapper) {
  return function() {
    switch (arguments.length) {
      case 0: return wrapper.call(this, func);
      case 1: return wrapper.call(this, func, arguments[0]);
      case 2: return wrapper.call(this, func, arguments[0], arguments[1]);
      // repeat x times...
    }
    // fallback
    var args = [func];
    push.apply(args, arguments);
    return wrapper.apply(this, args);
  }
};
```

This optimized function now runs 2-10 faster. In most cases there are no more intermediary objects created and `apply` has been replaced by the much faster `call`.

The downside is an increase of 70 bytes in minifed gzipped file size. However we can spread these bytes by reusing the fast method dispatching for other methods that use `apply`.

<a name="extract_for_reuse" /></a>

### Extract for reuse

As above function is used in other places as well we extract that into a separate function.

```javascript
function fastApplyContextPrepend(func, context, prependObj) {
  function () {
    var a = arguments;
    switch(a.length) {
      case 0: return func.call(context, prependObj);
      case 1: return func.call(context, prependObj, a[0]);
      case 2: return func.call(context, prependObj, a[0], a[1]);
      case 2: return func.call(context, prependObj, a[0], a[1], a[2]);
      // repeat x times..
    }
    // fallback
    var args = [prependObj];
    push.apply(args, arguments);
    return wrapper.apply(context, args);
  }
}
```

The inner function `_.wrap` passes on the unchanged `arguments` to `fastApplyContextPrepend`.

```javascript
_.wrap = function(func, wrapper) {
  return fastApplyContextPrepend(wrapper, arguments, this, func);
};
```

Result: [A cool 200% - 2000% improvement](http://jsperf.com/wrap-optimized)

<a name="fastApply" /></a>

### fastApply()

If you only want to call a method with an arguments object or an array the overhead of `call` can be avoided by invoking the function directly.

```javascript
function fastApply (fn, args, context) {
  if (context === undefined) {
    switch(args.length) {
      case 0: return fn();
      case 1: return fn(args[0]);
      // ...
    }
    return fn.apply(null, args);
  } else {
    // same as above but with call.
  }
}

function noop() { }

noop.apply(null, []);
fastApply(noop, []);  // 2-10 times faster than above
```

Just by avoiding `apply` we get a significant boost; [2-10 x faster](http://jsperf.com/custom-apply). And it works just like apply.


### apply is everywhere

It is important to note that `apply` is not just used for some exotic functions like `_.wrap`. There's the `_.mixin` which is used by underscores OO-style `_(obj)`. These all could profit from above optimizations. However they might need variations of above applyFast methods.

## Recap










