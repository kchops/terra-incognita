---
title: Intermezzo I - A C Interop Library
category: chapters
layout: post
order: 3
---

Here, as an example of a small real-world program, we are going to develop a minimal C interop library. Our aim is to be able to develop C libraries using Terra. We will. in principle, support:

- Generating a C header from a .t Terra file
- Generating a .so dynamic library from this .t Terra file

Initially, our implementation will only support exporting structs and functions from our Terra files. A production-grade version of this will have to handle all the edge cases and should use the reflection API fully. Nonetheless, the ideas will be same for making this example into a full-fledged C interop library.

This way, we can develop C libraries from Terra. Since C is the base language that every other language can call into, this will let us resuse our Terra code from any programming language possible.

Here is our directory structure:

```
|-math (dir)
    |-sin.t
|-cheader (dir)
    |-generator.t
|-cexport.t
```

Inside the `cexport.t`, we will have:

``` lua
local math = require("math.sin")
local cheader = require("cheader.generator")

cheader.export(math, "./out", "gcc")
```

You can run this as usual:
> $ terra export.t

Inside the directory `math`, we have a Terra file `sin.t` that we want to export to C:

``` lua
local cmath = terralib.includec("math.h")
local math = {}

struct math.Complex {real: double, imag: double}

terra math.sin(a: math.Complex) : double
    return cmath.sin(a.real)
end

return math
```

It contains a struct and a function, barebones that we support.

And inside the directory `cheader`, the file `generator.t` implements all the logic needed for the function:

`cheader.export(math, "./out", "gcc")`

We will give the contents of the file `generator.t` in full here, so that you can recreate the example yourself. Then, we will explain it.

``` lua
local module = {}

local module_name = ""

function string:split(sep)
   local sep, fields = sep or ":", {}
   local pattern = string.format("([^%s]+)", sep)
   self:gsub(pattern, function(c) fields[#fields+1] = c end)
   return fields
end

local function header_pre() 
    local s = ""
    s = s ..
        "#pragma once\n"..
        "#ifdef __cplusplus\n"..
        "extern \"C\" {\n"..
        "#endif\n\n"
    return s
end

local function header_end() 
    local s = ""
    s = s .. "\n" ..
        "#ifdef __cplusplus\n"..
        "}\n"..
        "#endif\n"
    return s
end

local function convert_c_func(key, value)
    local name = tostring(value)
           :split("\n")[1]
           :split(" ")[3]
           :split("%(")[1]
           :gsub("%.", "_")
    module_name = name:split("_")[1]
    local t = tostring(value:gettype()):split("->")
    local args = t[1]
    args = args:gsub("%{", "%("):gsub("%}", "%)")
    local ret = t[2]
    local cfunc = (ret.." "..name..args..";"):gsub("%.", "_")
    return cfunc:sub(2, cfunc:len())
end

local function convert_c_struct(key, value)
    local name = tostring(value):gsub("%.", "_")
    module_name = name:split("_")[1]
    local sinside = ""
    local N = 0
    for k, v in pairs(key.entries) do
        N = N +1
    end
    for i=1,N do
        sinside = sinside..
                  "\t"..
                  tostring(key.entries[i]["type"])..
                  " "..
                  tostring(key.entries[i]["field"])..
                  ";\n"
    end
    local s = "typedef struct {\n"..
              sinside..
              "}"..
              tostring(value):gsub("%.", "_")..
              ";\n"
    return s
end

local function get_type(value)
    if terralib.types.istype(value) then
        return value
    else
        return value:gettype()
    end
end

function make_header(m)
    local file_contents = ""
    for key, value in pairs(m) do
        local t = get_type(value)
        if t:isstruct() then
            file_contents = file_contents..
                            convert_c_struct(t, value)
        end
    end
    file_contents = file_contents.."\n"
    for key, value in pairs(m) do
        local t = get_type(value)
        if t:isfunction() then
            file_contents = file_contents..
                            convert_c_func(key, value)
        end
    end
    file_contents = file_contents.."\n"

    local hfile = io.open (module_name..".h", "w")
    io.output(hfile)
    io.write(header_pre())
    io.write(file_contents)
    io.write(header_end())
    io.close(hfile)
end

function make_func_table(m)
    local func_table = {}
    for key, value in pairs(m) do
        local t = get_type(value)
        if t:isfunction() then
            local name = tostring(value)
               :split("\n")[1]
               :split(" ")[3]
               :split("%(")[1]
               :gsub("%.", "_")
            func_table[name]=value
        end
    end
    return func_table
end

--[[
m: Terra module to be exported as C header and dynamic lib
dir: where to pur resulting files
compiler: C compiler used when compiling dynamic lib

example usage:
    cheader.export(math, "./out", "gcc")
]]--
function module.export(m, dir, compiler)
    make_header(m)
    terralib.saveobj(module_name..".o", 
                     make_func_table(m),
                  {"-fPIC"})

    os.execute(compiler..
               " -shared -fPIC -o "..
               "lib"..module_name..
               ".so *.o")
    os.execute("rm *.o")
    os.execute("mv "..module_name..".h".." "..dir)
    os.execute("mv ".."lib"..module_name..".so".." "..dir)
end

return module
```

As you can see, this is the biggest program we wrote so far. We will explain it fully. First thing to note, all the functions inside `generator.t` are pure Lua functions. We want to write these at compile-time, as a metaprogram. Main driver function, the only one we export from the module:

``` lua
function module.export(m, dir, compiler)
    make_header(m)
    terralib.saveobj(module_name..".o", 
                     make_func_table(m),
                  {"-fPIC"})

    os.execute(compiler..
               " -shared -fPIC -o "..
               "lib"..module_name..
               ".so *.o")
    os.execute("rm *.o")
    os.execute("mv "..module_name..".h".." "..dir)
    os.execute("mv ".."lib"..module_name..".so".." "..dir)
end
```

First line creates the header for the module `m`, we will explain this later. Then we call `make_func_table(m)` when calling saveobj. This will create a .o object file for us from the functions found in the result table of the `make_func_table(m)` call. We pass `-fPIC` flag to make it Position-Independent-Code, because we are going to create a dynamic C library from this object file.

``` lua
os.execute(compiler..
           " -shared -fPIC -o "..
           "lib"..module_name..
           ".so *.o")
```

This call basically turns into `gcc -shared -fPIC -o libmath.so *.o`. Then we do some cleaning, put the resulting header and dynamic library into desired directory etc. Then you can use these in your C projects. Let us look into `make_header`:

``` lua
function make_header(m)
    local file_contents = ""
    for key, value in pairs(m) do
        local t = get_type(value)
        if t:isstruct() then
            file_contents = file_contents..
                            convert_c_struct(t, value)
        end
    end
    file_contents = file_contents.."\n"
    for key, value in pairs(m) do
        local t = get_type(value)
        if t:isfunction() then
            file_contents = file_contents..
                            convert_c_func(key, value)
        end
    end
    file_contents = file_contents.."\n"

    local hfile = io.open (module_name..".h", "w")
    io.output(hfile)
    io.write(header_pre())
    io.write(file_contents)
    io.write(header_end())
    io.close(hfile)
end
```

Our parameter `m` is just a Lua table holding the Terra constructs we exported from our module. Thus, we iterate over this as `key, value`. I suggest printing these while running them such that you see the form in which Terra stores these. This way, the way we translate these into C constructs will be much clearer. We build the header file into an empty string, `file_contents`. We first go over all the Terra constructs in the module, take the ones that are of type `struct`. Convert them to C structs. Then we do the same for functions. While parsing these, we also parse the module name into a file-local variable called `module_name`. Inside our `sin.t`, we named our module `math` so this will be parsed into above variable. Then we do:

``` lua 
local hfile = io.open (module_name..".h", "w")
io.output(hfile)
io.write(header_pre())
io.write(file_contents)
io.write(header_end())
io.close(hfile)
```

This will create `math.h` file and put the things we parsed into the header. You need to inspect the rest of the functions, play with them, print the intermediate results to see how things work, maybe extend the implementation to support more than functions and structs etc. This is both an example and an exercise.

I use the resulting header and library as follows:

- I create a main.cpp file that uses the header:
   
``` cpp
   #include "math.h"
#include <stdio.h>

int main() {
    math_Complex c;
    c.real = 1.578;
    c.imag = 3.14;
    printf("%f\n", math_sin(c));
}
```

I can do this because we create the header correctly for  C++ as well, using `extern C`.

- Compile it like:
   
> $ g++ -o main main.cpp -L. -lmath -lm
because we internally use C's `math.h` header, we link `-lm`.

- Run ./main:
   
> ---- output ----

> 0.999974

Happy hacking!
