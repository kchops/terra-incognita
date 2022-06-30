---
title: Brothers in Arms; Terra, Lua and C
category: chapters
layout: post
order: 2
---

## Terra talking to Terra

OK, this is not language interop. What I mean is to investigate the ways of sharing Terra code between modules. Suppose we have a `main.t` file in a directory. Now we are going to implement a _math_ module. Create a directory called `math` in your working directory and add a file called `sin.t`.

```
|-math (dir)
    |-sin.t
|-main.t
```

Inside `sin.t`, let's start with the following:

``` lua
local cmath = terralib.includec("math.h")
local cstdio = terralib.includec("stdio.h")
answer = 42

local terra print_arg(arg: double) : {}
    cstdio.printf("%f\n", arg)
end

terra sin(a: double) : double
    print_arg(a)
    return cmath.sin(a)
end
```

and inside `main.t`:

``` lua
local cstdio = terralib.includec("stdio.h")
require("math.sin")

terra main(argc : int, argv : &rawstring)
    cstdio.printf("sin of pi/6=%f\n", sin(0.523))
    cstdio.printf("Answer=%d\n", answer)
    -- print_arg(0.523)
    -- cmath.sin(0.523)
    return 0
end

terralib.saveobj("main", { main = main }, {"-l","m"})
end
```

Let's start with `sin.t`. The first two lines are familiar to us by now, usual way of including C headers. Then we create a Lua global variable, `answer`. We could have made this local too. This is the outer context so we are just doing regular Lua programming. On the next line, we create a Terra function but a little bit different this time:

`local terra print_arg(arg: double) : {}`

This function is marked `local` so it won't be visible for files importing `sin.t` module. This is apparent in the commented-out line inside `main.t` saying: 

`-- print_arg(0.523)`

> Yeah, `--` is used for single line comments.

`print_arg` function has one parameter called `arg` of type `double`. It's return type is empty, C's `void`. This is signalled here as `:{}`. We could have just left it empty too, as in:

`local terra print_arg(arg: double)`

Implementation of the `print_arg` function is familiar to us at this point. This file also contains another Terra function, `sin`. Implementation of this should be understandable by now. 

If we go to the file `main.t`, it starts with some `includec` calls. Then we see something new, `require`.

`require("math.sin")`

This means "get the contents of the Terra file at ./math/sin.t". Of course, things that are marked with `local` inside the imported file will not be visible here at `main.t`. We will only see the function `sin(a: double)`. Notice the other commented out line in `main.t`:

`-- cmath.sin(0.523)`

`cmath` import is also local inside `sin.t` and won't be visible here at `main.t`. If we commented out these problematic lines, our program would not compile. Notice that Lua variable `answer` from `sin.t` is globally visible inside `main.t` because we did not make it local.

Rest of the `main.t` requires no explanation. Just one part maybe, when we create our executable with the call:

`terralib.saveobj("main", { main = main }, {"-l","m"})`

Here we link the _libm_ math library. This is required on some Unix systems when using the header `math.h`.

You can read the [documentation](https://terralang.org/api.html#loading-terra-code) for `require` here.

## Basics of Terra - C Interop

We saw `includec` for including C headers. Since we compile with LLWM when using Terra, we also saw linking libraries that were available on library path (LD_LIBRARY_PATH on Linux, as example). This already gives us very good C interop. Still, Terra offers more.

Here is a standalone example that we can place inside `main.t`:

``` lua
local libuv = terralib.includecstring [[
    #include <uv.h>
    #include <stdio.h>

    void print_sys() {
        uv_utsname_t uname;
        uv_os_uname(&uname);
        printf("%s\n", uname.sysname);
    }
]]

terra main(argc : int, argv : &rawstring)
    --[[
        Do you want to see a multiline comment?
        This is one!
    ]]--
    libuv.print_sys()
    return 0
end

terralib.saveobj("main", { main = main }, {"-l","uv"})
```

`includecstring` function of `terralib` offers another way of interfacing with C. Here, as you can see, we just write our C code. Then with the help of `includestring` function, type definitions and functions from our C code is converted into Terra types and function, then is places inside a Lua table. We call our table `libuv` here. In our C code, we include `<uv.h>`. This is the header for the famous Async IO library _libuv_. Since I installed this with a package manager on my system, includes are available on default system include paths. If this was not the case, `includecstring` has options to pass flags to underlying LLVM toolchain such that we can add new include directories. 

Then we call our C function from Terra code as `libuv.print_sys()`. This function is like the shell comman `uname` and prints `Linux` on my system.

Notice again how we link _libuv_ with the `saveobj` call. Also as a sidenote, observe our multiline comment usage.

C interop documentation is [here](https://terralang.org/api.html#using-c-inside-terra).

> I have to note that on my system, the `terralib` function `linklibrary` only works if I give full-paths for the dynamic library .so files.

## Terra - Lua Interop

Terra uses LuaJit as the embedded Lua system. LuaJit comes with a powerful FFI that translates between C types and Lua types. Since Terra is pretty close to C with respect to type system, Terra reuses this LuaJit FFI to translate between Lua and Terra types and functions.

Documentation is pretty good on this topic. [Lua-Terra Interaction](https://terralang.org/getting-started.html#lua-terra-interaction) is described here. Note that this is a pretty big topic, it is the mean mechanism for metaprogramming in Terra since we use Lua to sort of script Terra programs. Here, we will just go over some fundamentals from the above documentation. 

Let us inspect the example from the above doc pages:

``` lua
struct A { a : int, b : double }

terra foo(a : A)
    return a.a + a.b
end

assert( foo( {a = 1,b = 2.3} )== 3.3 )
assert( foo( {1,2.3} ) == 3.3)
assert( foo( {b = 1, a = 2.3} ) == 3 )
```

Here, we first define a Terra struct type called `A`. Inside the `assert` call for example, see how Lua tables are converted directly to Terra structs. In the last `assert`, order of members is changed and `a:int` member is given a float so it is floored to 2 automatically. We should keep such implicit conversions in mind, it is a tricky feature found in C as well. All in all, this example shows the translation of values between Lua and Terra. Complete set of conversion rules can be found on the linked documentation. 

Above example shows a Terra function, `foo`, being called from Lua (outermost context is basically Lua programming). Next example on this portion of the docs shows calling Lua functions from Terra:

``` lua
function add1(a)
    a.real = a.real + 1
end

struct Complex { real : double, imag : double }

tadd1 = terralib.cast({&Complex}->{},add1)

terra doit()
    var a = Complex {1,2}
    tadd1(&a)
    return a
end

a = doit()
print(a.real,a.imag) -- 2    2
print(type(a)) -- cdata
```

Here, `add1` is pure-Lua function. Normally, you would call this with a table having a key `real` etc. Following call:

`tadd1 = terralib.cast({&Complex}->{},add1)`

casts this Lua function to a Terra function which takes a pointer to type `Complex` and returns nothing. 

> Yes, pointer to type T is denoted &T on Terra.

Now we can use this casted function `tadd1` from other Terra functions, in this case, the Terra function `doit`. It is important to see that the type of `a` is now `cdata`. This is result of LuaJIT FFI conversion applied for Lua-Terra interop.

The most important part of Lua-Terra interop is explained [here](https://terralang.org/getting-started.html#meta-programming-terra-with-lua) in the docs. This concerns metaprogramming Terra using Lua.

Our standalone example that can be put inside a `main.t` is program to generate accessor functions for types. This is probably not a good way to access values of the fields of your aggregates like structs but it is a pedagocical example anyways.

``` lua
local cstdio = terralib.includec("stdio.h")

struct Employee { age : int, name : rawstring }
struct Building { age : int, street : rawstring }

function make_getter(Type, field)
    local terra field_(t: Type)
        return t.[field]
    end
    return field_
end

terra main()
    var employee_age_getter = [make_getter(Employee, "age")]
    var e = Employee{age = 42, name = "John"}
    cstdio.printf("%d\n", employee_age_getter(e))

    var building_street_getter = [make_getter(Building, "street")]
    var b = Building{age = 40, street = "Charles De Gaul"}
    cstdio.printf("%s\n", building_street_getter(b))
    -- compile error
    --cstdio.printf("%d\n", employee_age_getter(b))
    return 0
end

make_getter(Building, "street"):printpretty()

terralib.saveobj("main", { main = main })
```

`make_getter` is a Lua function. You can pass Terra types as arguments to Lua functions because at compile-time, Terra types are just regular values for Lua. Later on we will see that this mechanism is also used to implement functionality like C++ templates. Anyways, the `Type` is just going to be a Terra type. The other argument `field` is, for our purposes, going to be a string literal like `"age"`. Then this function will return a Terra function, which eventually will return `field` from an object of type `Type`. The returned function is implemented as:

``` lua
local terra field_(t: Type)
    return t.[field]
end
```

We then return this `field_` function, as you saw. This function takes t, an object of type `Type`. To return the `field` member, we see a curious construct:

`return t.[field]`

This is what is known as _escapes_ in Terra. Escapes let you insert Lua code into your Terra code as tokens. So Lua code `t.["age"]` turns into `t.age` when regarded as Terra code.

We see another use of escapes in the beginning of the `main`:

`var employee_age_getter = [make_getter(Employee, "age")]`

This will return the name of a Terra function but the function's implementation will be modified inside the escape depending on the parameters we supply. Notice that we would not be able to pass a type like `Employee` to a function in Terra if this was not surrounded by `[]`. But escapes let's us execute this code as Lua code at compile-time.

The output of the following call let's us see what Terra function we get from the escape:

`make_getter(Building, "street"):printpretty()`

I get the output:

``` lua
main.t:17:              terra field_(t : Building) : &int8
main.t:18:                  return t.street
main.t:17:              end
42
Charles De Gaul
```

This is a function, as the name suggest, we can call to pretty print Terra constructs.

Another why of Lua-Terra interop is called _quotes_. This is used to generate Terra expressions or statements in programmatic way from Lua. Another example `main.t`:

``` lua
local cstdio = terralib.includec("stdio.h")

struct point_2d { x : float, y : float }

function hypotenuse(p)
    return `p.x * p.x + p.y * p.y
end

terra main()
    var m = point_2d{x = 3.0, y = 4.0}
    cstdio.printf("%f\n", [hypotenuse(m)])
    return 0
end

terralib.saveobj("main", { main = main })
```

Quotation happens in our Lua function `hypotenuse`. Quotes are a way to generate expressions and statements. Once you have your quotes, you can use escapes to compose them to your heart's content. Here, we basically metaprogram the call:

`cstdio.printf("%f\n", m.x * m.x + m.y * m.y)`

Of course, here it was a very contrived example. But keep in mind we can pretty much construct any expression. In turn, we can, for example build very complex mathematical expressions while avoiding the overhead of function calls. These might also be better suited for optimizations, compared to function calls. It is basically programmable-inlining. Such a technique is used in performant C++ math libraries like Eigen.

Of course, ` form is used for expressions and due to Lua handling expressions-statements separately, we need a different system for conjuring quoted statements. There we use the `quote` keyword.

``` lua
local cstdio = terralib.includec("stdio.h")

function log(msg)
    return quote
        cstdio.printf("%s\n", msg)
    end
end

logger = macro(
    function(msg) 
        return quote
            cstdio.printf("%s\n", msg)
        end 
    end
)

terra main()
    var x : int = 0
    var y : int = 42
    if x == 0 then
        [log("Do no divide by 0")]
        logger("Do no divide by 0")
        return 0
    else
        return y / x
    end
end

terralib.saveobj("main", { main = main })
```

Of course, we could definitely implement better logging mechanism but this demonstrates conjuring statements programmatically. We have to point out again, definitely carefully read [this](https://terralang.org/getting-started.html#meta-programming-terra-with-lua) part of the documentation. Here, we show two ways of using quote to conjure statements; writing Lua functions to be called inside escapes and creating `macros`.

One point we might want to go over again is macro _hygiene_, explained in the docs nicely. We will recrate the example from the docs here:

``` lua
function makeexp(arg)
    return quote
        var a = 2
        return arg + a
    end
end

terra client()
    var a = 1;
    [ makeexp(a) ];
end
```

It is helpful to compare this with an _unhygienic_ metaprogramming system, like C preprocessor. This is also given in the docs:

``` c
#define MAKEEXP(arg) \
    int a = 2; \
    return arg + a; \

int scoping() {
    int a = 1;
    MAKEEXP(a)
}
```

As you know, the expanded C code will be like:

``` c
int scoping() {
    int a = 1;
    int a = 2;
    return a + a;
}
```

The macro expansion will litter the calling function's scoping environment, ie. it will mess with the definition of `a`. This is called an _unhygienic_ macro system. Terra metaprogram will keep track of the different versions of a and will correctly expand into `return 3`. Terra respects scoping environment of calling function, its metaprogramming system is _hygienic_.

We will explore all these features further as we write more and more real-world example programs in upcoming chapters. We end this chapter with one such example program.

## Keyword Arguments in Function Calls

Suppose you have a Terra function like `foo(a: int, p: double)`. We call this function naturally something like `foo(42, 3.14)`. In some languages, they support _keyword arguments_ so a function can also be called like `foo(a=42, p=3.14)`. We want to implement a simple version of this for Terra functions using metaprogramming.

``` lua
local cstdio = terralib.includec("stdio.h")

local function kw_call_wrapper(fn)
    return macro(
        function(arg_struct)
            local arg_values = fn.definition.parameters:map(
                function(p) 
                    return `(arg_struct.[p.name])
                end
            )
            return `fn(arg_values)
        end
    )
end

terra foo(a : int, p : double)
  cstdio.printf("a %d p %f\n", a, p)
end
foo = kw_call_wrapper(foo)

terra main()
  foo({ a = 42, p = 3.14 })
  -- can be called like foo { a = 42, p = 3.14 } 
end

terralib.saveobj("main", { main = main })
```

At the call site, our solution requires some additional syntax with `{}` since we use Lua tables to store keyword arguments, but other than that, everything is quite succint and clear.

First, this is definitely going to be a metaprogram but we also want to be able to call it like a function, so we make a macro. What happens? `kw_call_wrapper(foo)` takes our function and returns a macro. You can clearly see this in the definition of `kw_call_wrapper`, as in `return macro`. This macro then called as we saw:

`foo({ a = 42, p = 3.14 })`

by passing a Terra struct/tuple. Name of the parameter for our macro suggests this. 

`function(arg_struct)`

Inside the macro, we basically create a Terra `List` of values to be passed into our function as:

``` lua
local arg_values = fn.definition.parameters:map(
    function(p) 
        return `(arg_struct.[p.name])
    end
)
```

Here, we use some introspection API from Terra. It lets us map over the values of function's parameters. `parameters`, as is Terra itself implemented, also a Terra `List` of values. So when we `map`, return value will be another `List`. What would be the values in the resulting `List`? We will get the values for parameters from our `arg_struct` as:

`arg_struct.[p.name]`

Remember, this macro will be "compile-time". So if our function has a parameter called `a` for example, above escape will create Terra code like: 

`arg_struct.["a"]`

and this will eventually become

`arg_struct.a`

But notice, our map is actually returning a `List` of quotes.

`return `(arg_struct.[p.name])`

This is needed because `arg_struct` is a Terra construct. Our macro is pure-Lua happening at "compile-time". So any Terra code found in macros has to be quoted and delayed. This example is a little more strange though; Terra code we run from our Lua macro is inturn calling Lua from Terra using an escape, `[p.name]`.

So rule is, when metaprogramming, Terra code from Lua needs to be quoted and Lua code called from Terra needs to be escaped. Anyways, when we finally have this `List` of values as quotes, this is then passed to another quote:

`return `fn(arg_values)`

which returns from our macro an expression that calls our initial function with the parameters coming from the struct passed to our macro. In short:

`foo({ a = 42, p = 3.14 })`

becomes

`foo(42, 3.14)`

The fact that Terra's function parameters are just Lists can be shown as follows:

``` lua
local List = require("terralist")
local cstdio = terralib.includec("stdio.h")

terra foo(a : int, p : double)
  cstdio.printf("a %d p %f\n", a, p)
end

ar = List {42, 3.14}

terra main()
  -- or call as foo([ar]) as escape is implicit in this context because this
  -- Lua list can be converted to a Terra value easily
  foo(ar)
end

terralib.saveobj("main", { main = main })
```

Well, Terra's `List` is actually just a Lua table with additional methods such that it is easier to metaprogram. You can see docs for [List](https://terralang.org/api.html#list) here. `map` we used on our macro was just one of the examples Terra's `List` provide over plain Lua tables. Nonetheless, this fact that function parameters are implemented in terms of `List` values was the backbone of our keyword argument implementation. Of course, we also mentioned using some introspection API from Terra. You can see here what type of [reflection](https://terralang.org/api.html#function) capabilities are available for functions.































