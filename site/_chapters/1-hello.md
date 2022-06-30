---
title: Hello, World!
category: chapters
layout: post
order: 1
---

This is where all the fun in programming begins, writing this canonical programming example in your choice of language. Hence, we are going to go over a simple "Hello, World!" program in Terra here.

Put following inside a file like _main.t_.

``` lua
local cstdio = terralib.includec("stdio.h")

terra hello(argc : int, argv : &rawstring)
    cstdio.printf("Hello, World!\n")
    return 0
end

terralib.saveobj("main", { main = hello })
```

Then do:

```
$ terra main.t
$ ./main
```

to finally tackle the most important stage of learning any programming language.

Still, even this basic Terra program beneifts from line-by-line explanation. Let's go!

`local cstdio = terralib.includec("stdio.h")`

This part actually shows one of the most fascinating features of Terra, its integration with the C language. Terra language is actually available to us with all its power at compile-time and at run-time. We wield this power using the library `terralib`. We call the `includec` function to use the standard C header `stdio.h`. This will return us a regular Lua table which then is assigned into module-local Lua variable `cstdio`. Yes, this is actually just Lua code yet. Next line will start the Terra code part.

`terra hello(argc : int, argv : &rawstring)`

This is how we introduce Terra code into our programs, we use the keyword `terra` to write a Terra function. As we said, the things we executed on the outermost scope was actually just Lua code, which is hugely Terra interoperable anyways. But now we started a Terra function and we delimit it with a matching `end` line when we are done.

`cstdio.printf("Hello, World!\n")`

As we said, `cstdio` is just a Lua table holding the funcfions coming from our header, `stdio.h` in this case. We can use it to call `printf` and this is exactly the C printf function. For example we can say `cstdio.printf("Ultimate Answer=%d\n", 42)` to format and all that jazz. 

Next line, we `return 0` as the customary way of ending successful programs. After that we close our Terra function definition with `end`.

Then comes something else, something again unique to Terra.

`terralib.saveobj("main", { main = hello })`

Again we take advantage of the fact that Terra compiler is available to us at runtime and harness the library `terralib`, this time `saveobj` function. This will let us control how our program should be compiled. 

This line says I am going to compile an executable called _main_, the first argument. The next argument `{ main = hello }` says Terra function `hello` will be the entry point of our program. 

This way, we won't need any seperate build system. Our programs themselves will know how they should be compiled and built.

Finally, we can run this program using regular `./main`.

Most of these points will be explored further. `terralib` itself is insanely powerful; you can dynamically compile new functions at runtime, change program JIT behaviour etc. Underlying these, Terra uses LLVM for compilation. Just as a teaser, `saveobj` could have been used like the following:

`terralib.saveobj("main", { main = main }, {"-l","uv"})`

Assuming _libuv_ Async IO library is installed on your system and is available in your linker library path, this will link _libuv_ into your program. When we used `stdio.h`, we did not have to link with anything because LLVM tools link the C Standard Library by default.

Or furthermore, you will see how Lua-Terra interop enables powerful metaprogramming capabilities. This was just the tip of the iceberg.