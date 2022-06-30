---
title: Intermezzo II - Mixins for Dynamic Dispatch
category: chapters
layout: post
order: 4
---

Suppose we want to develop a video compression library. We will have objects in this library offering following functionality:

``` lua
compress: {rawstring, int} -> {bool}
play: {} -> {}
```

Do not focus heavily on design here, these functions does not have to make sense. `compress` will take a path as `rawstring`, a desired bitrate as `int`, then it will return a `bool` indicating if the compression was successful. `play` will just send out this compressed video data through some medium like TCP/IP. It takes no argument and returns nothing.

In popular OOP languages, as you know; you first make an abstract class/interface kind of thing for these set of functions, then you make your object inherit/extend etc. from this interface.

Another approach to runtime polymorphism is what is known as _mixins_. This will be better explained with an example. Suppose we have following code in our Terra file:

``` lua
local compress_msg = dyno.dynamic_msg({int}, {rawstring, int})
local play_msg = dyno.dynamic_msg({}, {})

local video_service =  dyno.dynamic_object({
                            compress = compress_msg, 
                            play = play_msg
                       })
```

`dyno` is the name of our Terra module, hinting to _dynamic objects_ in the name. As you can see, we described our desired API with `dynamic_msg` calls and then we described a dynamic object (this returns a type) abstractly, fulfilling this API.

Suppose somewhere (or maybe in the same file, definitely in the same file in this example program) someone else wrote following code already:

``` lua
struct mpeg_compression {
    major_version: int,
    minor_version: int
}

terra mpeg_compression:compress(path: rawstring, rate: int) : int
    cstdio.printf("Compressing the video file at %s\n", path)
    cstdio.printf("compression algorithm=mpeg\n")
    cstdio.printf("compressing to %d mpbs\n", rate)
    cstdio.printf("mpeg version %d.%d\n", 
                  self.major_version,
                  self.minor_version)
    return 43
end

terra mpeg_compression:change_psi()
    cstdio.printf("changing PSI info\n")
end

struct xs_compression {
    ptp_time: int
}

terra xs_compression:compress(path: rawstring, rate: int) : int
    cstdio.printf("Compressing the video file at %s\n", path)
    cstdio.printf("compression algorithm=xs\n")
    cstdio.printf("compressing to %d mpbs\n", rate)
    cstdio.printf("ptp %d\n", self.ptp_time)
    return 42
end

struct ip_playout {
    ip_addr: rawstring
}

terra ip_playout:play()
    cstdio.printf("Playing the video to ip %s\n", self.ip_addr)
end

struct sdi_playout {
    port: int
}

terra sdi_playout:play()
    cstdio.printf("Playing the video to SDI port %d\n", self.port)
end
```

As you can see, someone already implemented these functionality in pieces. Maybe MPEG compression comes from a different library and IP playout from a different library, that's possible. If we used inheritance to get dynamic polymorphism, we would need to modify types coming from a 3rd party library such that those are extending/inheriting our abstract interfaces etc. Here, mixins won't ne intrusive like this. We can use any type we want in out dynamic objects.

Here is how we use this dynamic object/mixin idea:

``` lua
terra main(argc : int, argv : &rawstring)
     -- our dynamic object, obj of this type can be stored in containers
    var vs : video_service[2]

    dyno.add_mixin(vs[0], "compress", mpeg_compression)
    dyno.add_mixin(vs[0], "play", ip_playout)
    dyno.getconcretetype(vs[0], "play", ip_playout).ip_addr = "127.0.0.1"
    dyno.getconcretetype(vs[0], "compress", mpeg_compression).major_version = 4
    dyno.getconcretetype(vs[0], "compress", mpeg_compression).minor_version = 2
    vs[0]:compress("/usr/bin/vid.mpeg", 40)
    vs[0]:play()

    dyno.add_mixin(vs[1], "compress", xs_compression)
    dyno.getconcretetype(vs[1], "compress", xs_compression).ptp_time = 753453453
    vs[1]:compress("/usr/bin/vid.mpeg", 50)

    -- change compression behaviour at runtime
    dyno.add_mixin(vs[1], "compress", mpeg_compression)
    dyno.getconcretetype(vs[1], "compress", mpeg_compression).major_version = 4
    dyno.getconcretetype(vs[1], "compress", mpeg_compression).minor_version = 2
    vs[1]:compress("/usr/bin/vid.mpeg", 50)
end
```

We can mix and match parts of our dynamic object from any type that implements a function with desired signature. We can change parts of it at runtime. Here, we use the type referring to our dynamic object `video_service`, in fact, an array of them. Then we use relevant parts of our `dyno` module; `add_mixin` and `getconcretetype`. Then we can call the methods `compress` and `play` as if these methods were available on our dynamic objects. Here is the output from above program.

```
Compressing the video file at /usr/bin/vid.mpeg
compression algorithm=mpeg
compressing to 40 mpbs
mpeg version 4.2
Playing the video to ip 127.0.0.1
Compressing the video file at /usr/bin/vid.mpeg
compression algorithm=xs
compressing to 50 mpbs
ptp 753453453
Compressing the video file at /usr/bin/vid.mpeg
compression algorithm=mpeg
compressing to 50 mpbs
mpeg version 4.2
```

Now we come to explaining the code for `dyno` module. We first give the code in full:

``` lua
local cstdlib = terralib.includec("stdlib.h")

local module_ = {}

function string:split(sep)
   local sep, fields = sep or ":", {}
   local pattern = string.format("([^%s]+)", sep)
   self:gsub(pattern, function(c) fields[#fields+1] = c end)
   return fields
end

function make_mixin(Type, fn)
    local symbols = terralib.newlist()
    local t = Type:astype()
    local args = t.methods[fn]:gettype().parameters
    for i=2,#args do
        symbols:insert(symbol(args[i]))
    end
    local terra mixin_(o:&opaque, [symbols])
        return [&t](o):[fn]([symbols])
    end
    mixin_:setinlined(true)
    return mixin_
end

function module_.dynamic_msg(Ret, args)
    local t = {}
    t[1] = &opaque
    for i=1,#args do
        t[i + 1]=args[i]
    end
    return t -> Ret
end

function module_.dynamic_object(msg_types)
    local newType = terralib.types.newstruct()
    for k, v in pairs(msg_types) do
        newType.entries:insert({field = k, type = v})
        newType.entries:insert({field = "_"..k.."_", type = &opaque})
    end
    for k, v in pairs(msg_types) do
        local symbols = terralib.newlist()
        for i=2,#v.type.parameters do
            symbols:insert(symbol(v.type.parameters[i]))
        end
        newType.methods[k] = terra(self: &newType, [symbols])
            return self.[k](self.["_"..k.."_"], [symbols])
        end
    end
    return newType
end

module_.add_mixin = macro(function (dy, msg, handler)
    local ht = handler:astype()
    local ms = msg:asvalue()
    ms = ms:sub(1,ms:len())
    return quote
        if not (dy.["_"..ms.."_"] == nil) then
            cstdlib.free(dy.["_"..ms.."_"])
        end
        dy.["_"..ms.."_"]=[&opaque](cstdlib.malloc(sizeof([ht])))
        dy.[ms] = [make_mixin(handler, ms)]
    end
end)

module_.getconcretetype = macro(function (dy, msg, type)
    local t = type:astype()
    local ms = msg:asvalue()
    ms = ms:sub(1,ms:len())
    return `[&t](dy.["_"..ms.."_"])
end)

return module_
```

Notice that this is an example implementation. Memory safety and error handling part is completely overlooked. Nontheless, we can start explaining this code.