---
layout: home
---

## Where are we?

Cartographers use the Latin expression **Terra Incognita** on maps to indicate uncharted lands and seas. There also lies the greatest adventures.

With a similar spirit, we are going to discover the programming language Terra. Our journey will start here at the main [Terra Web Page](https://terralang.org/) and we will frequently visit this GitHub page where [Terra Source Code](https://github.com/terralang/terra) lives.

## Why are we here?

Terra is not a famous programming language, used in very few production-grade software, if any. Then why do we want to learn it? Well, I believe it is serving a very important but much overlooked need in the programming world.

Let's face it, the world runs on C and it is still _the programming language_ in the following sense; our software communicate with each other through C ABI. It is still a very good abstraction of the underlying hardware, as some people call it a _portable assembler_. Not just that, C is still a widely used systems programming language. It is also still popular in areas like game programming, embedded programming, numerical computing; basically places where you need to control the hardware resources tightly. 

It is no surprise then the other languages wanted to improve on the success of C in various ways. C++ wanted to keep C's lower-level control while improving on its abstraction capabilities, Java wanted to do that as well but sacrificed some lower-level control to get memory safety, Rust tried to do both without sacrificing any lower-level control etc. Examples are abundant.

One area is not very well explored though; while keeping programming model of C pretty much intact, improving on its metaprogramming capabilities. This means rather than improving C in this or that way, giving programmers the power to improve C in whatever way they see fit. 

Terra is doing just that. The language's runtime model is rather similar to C. But whereas C can only be poorly metaprogrammed using the preprocesser, Terra bakes in a full-fledged Lua environment (available at compile-time as well) for us to metaprogram our programs.

I believe this is an important paradigm to be explored. No mainstream programming language offers this capability (maybe Rust but it is not just C + metaprogramming, it is C + too many opinionated defaults + metaprogramming). Will Terra be ever mainstream? I would love to see that but it is a long shot, at least, it will require much more effort.  But the idea behind is very important, one worth exploring.

## Assumptions

I will assume you are well-versed in C programming and are computer-literate. For example, we will run our examples in an online programming environment called _replit_ but it will be up to you to troubleshoot it. Or if you want to install Terra to your local machine, it will be on your hands to complete everything. I will just point you to the resources. I will still try to note down Terra related stuff extensively though. Afterall, not many of us are Terra programmers. Even I am not one, this is a learning exercise for me too.

Let's start!
