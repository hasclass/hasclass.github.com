---
layout: hasclass
title: Javascript the Fast Parts
---

# Javascript the Fast Parts

While writing the RubyJS library I learned a lot about writing fast JavaScript code. JavaScript has become a very fast language in the recent years. But that doesn't mean you can't be sloppy with your code. Without benchmarking diligently you end up with crippling slow code. That means you have to look past just the 'good parts' of JavaScript, and embrace the little nitty gritty details. In this ongoing series I'll outline some of the techniques I've learned and applied to the RubyJS library.


## Articles

[Avoid intermediary objects and Function apply](/articles/avoid-intermediary-objects-and-apply.html)

## Should you micro optimize your code?

A good example is underscorejs vs lodash, the former does not optimize for speed but for clarity and lines of code. It's fast enough, and only when you need to go faster you should start optimizing, also you don't know what the future will bring. On the other hand there is lodash, which started as a fork of underscore, and adds numerous performance improvements (and bugfixes).

Personally I respect both views. Optimizing and micro-optimizing libraries is tedious, and can also be a moving target. However after a lot of benchmarking, I realized that most optimizations have one clear winner across old and new browsers with different engines. And these are mostly  significant improvements, like 2-20 times faster.

## Who should use these techniques

Generally speaking you shouldn't prematurely optimize your code for performnace. Especially your application logic/code where clarity and testability is more important. However library authors (which includes internal libraries or "helper"-functions) can profit by giving a performant base code.





