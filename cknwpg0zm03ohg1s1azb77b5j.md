## The case of the unwieldy HashMap

Some data structure was pinning over 70MB of heap space in Android Studio. Our developers have limited memory on laptops, and are often upset about memory consumption in general. This behemoth (retained by our internal plugins) was the second largest allocated and pinned single object in the whole of AS's heap.

[buck project](https://buck.build/command/project.html) generates IDE project definitions from buck targets. It can be configured it to emit a target-info.json file, which contains simple mappings that look something like this:

```lang=json
{
  "//src/com/foo/bar/baz:baz" : {
    "buck.type": "java_library",
    "intellij.file_path" : ".idea/modules/src_com_foo_bar_baz_baz.iml",
    "intellij.name" : "src_com_foo_bar_baz_baz",
    "intellij.type" : "module"
    "module.lang" : "KOTLIN",
    "generated_sources" : [ 
      "buck-out/fe3a3a3/src/com/foo/bar/baz/__gen__", 
      "buck-out/fe3a3a3/src/com/foo/bar/baz/__gen_more__" 
    ],
  },
  "//foo/far/fun:fun" : {
    "buck.type": "cxx_library",
    "intellij.file_path" : ".idea/modules/foo_far_fun_fun.iml",
    "intellij.name" : "foo_far_fun_fun",
    "intellij.type" : "module"
  }
}
```

We have some large number of these targets (ballpark 100k or so), so the existing datastructure representation of this (a hashmap correlating to the structure above) could become quite large. The datastructure was intentionally pinned in memory in case we needed it.

This was a fun small optimization problem. The map is accessed infrequently in bursts, and the number of keys we typically have to look up are a tiny proportion of the total set of keys in the map. Our ideal structure has relatively fast lookups with low memory overhead. Here are some of the things we might try:

#### Load the file lazily when we need to do a lookup

Instead of pinning this datastructure in memory permanently, try to arrange the code so that we load the file once into a HashMap, and use it locally where it's needed, allowing the HashMap to be garbage collected when we're done. 

Loading the file is relatively slow: it takes about 600ms to read and parse when the system's filesystem cache is cold, and about 200ms otherwise. But assuming we can arrange the code in a way where this happens only once, and it happens in a way that doesn't block anything else, that might be acceptable. 

This option turned out to be impractical, because of the architecture of the part of the plugin API in IntelliJ it was invoked from. The component which renders file inspections is recreated multiple times while a file is visible on screen, and there's no convenient way to attach the loaded HashMap to the context of an open editor. Given that, most lookups would take in the 200ms range, and this would generate large amounts of garbage to be collected on the heap, leading to increased GC times.

#### Optimize the in-memory representation of the data

There's a fair amount of repetitive pattern based naming in the original structure. For example, a target called //foo/bar maps to a module called foo\_bar. There are different configurable schemes for how targets map to module names, so the above mapping isn't necessarily canonical. We could consider the options used to generate the project information, and map back the names dynamically at runtime. 

I didn't extensively investigate this option. Logically, it'd reduce memory usage by about a third, and the extra processing required would likely be quite cheap.

#### Load the file lazily and hold it in a time-based cache

This is similar to the first option, except we alleviate the problem of not having a convenient place to keep hold of the HashMap by retaining it in a WeakReference cache for a fixed period of time after it's last used. I ended up implementing this first as a stopgap, because it's relatively easy. 

It does have the downside of still using a large amount of heap for some period of time after the last access, and potentially can also generate a lot of garbage to be collected depending on usage patterns.

#### Change the file to a binary format to speed up reads

The 600ms initial disk read time seems high. This file clocks in at 37MB, under ideal conditions on a 500 MB/s SSD, we'd expect to be able to read it in under 100ms. Indeed that matches what we see if we use dd to measure read performance:

```lang=bash
    # Purge disk buffers
    $ sudo purge
    
    # First read is about 200ms
    $ time dd if=target-info.json of=/dev/null bs=8k
    4740+1 records in
    4740+1 records out
    38837604 bytes transferred in 0.019201 secs (2022682285 bytes/sec)
    dd if=target-info.json of=/dev/null bs=8k  0.00s user 0.01s system 67% cpu 0.023 total
    
    # Second read, buffered in the disk cache, is about 100ms
    $ time dd if=target-info.json of=/dev/null bs=8k
    4740+1 records in
    4740+1 records out
    38837604 bytes transferred in 0.010184 secs (3813571762 bytes/sec)
    dd if=target-info.json of=/dev/null bs=8k  0.00s user 0.01s system 92% cpu 0.013 total
```

We should profile to see why there's such a large discrepancy between ideal and observed read time, but some theories about this:

*   Read contention
*   Overhead of parsing JSON
*   Cost of allocating object on the heap while building the map

Writing some code to use a raw binary format for the file clearly demonstrated that the JSON parsing wasn't the issue. I didn't profile it to my shame, but I think it's likely allocation is the primary contributor. Overall, it's very wasteful that we're allocating so many objects that we don't need.

#### Don't load the file into memory at all

Ideally, we'd just avoid reading this whole file into memory altogether. Since we're unlikely to use most of it, it's always going to be pretty wasteful. 

We could use some kind of mechanism to [stream the JSON](https://en.wikipedia.org/wiki/JSON_streaming) and find just the key we're looking for. But since we control the generation of the file, it seems like it might be better just to write out a file format that's makes it easy to look up a specific key and read data directly from the file for that key. A [B-Tree](https://en.wikipedia.org/wiki/B-tree) is a good datastructure for this kind of problem, but I wound up doing something much simpler to implement. The [next post](https://blog.dubh.org/2020/10/hashfile-disk-based-hash-structure.html) talks about a [disk-based hash structure](https://blog.dubh.org/2020/10/hashfile-disk-based-hash-structure.html) I used to solve this problem.