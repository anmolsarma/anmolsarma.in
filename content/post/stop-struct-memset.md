+++
title = "Stop Memsetting Structures"
date = 2019-04-27T23:59:42+05:30
draft = false 
tags = ['tech', 'C', 'rant', 'language-lawyering']

+++


*TL;DR: Use C99's designated initializers instead. Because it's 2019!*


*__Edit:__ This post was discussed on [Hacker News](https://news.ycombinator.com/item?id=19766930), [Lobste.rs](https://lobste.rs/s/7aqidd/stop_memsetting_structures) and [/r/c_programming](https://www.reddit.com/r/C_Programming/comments/bi23w6/stop_memsetting_structures/). It has been updated to address some of the concerns raised by commenters.*

One of the little things that annoy me when reading code is the unnecessary use of `memset()` to zero-out a `struct`. Something I see frequently in networking code that uses the BSD sockets API.

Consider this snippet of code from the venerable [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/html/single/bgnet.html#simpleserver):
```C
struct addrinfo hints;
...

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE; // use my IP
```

So, we first declare a struct, memset it to zero and then initialize the fields we are interested in. This is all fine except for the fact that the same result can be achieved with the following:
```C
struct addrinfo hints = {
	.ai_family = AF_UNSPEC,
	.ai_socktype = SOCK_STREAM,
	.ai_flags = AI_PASSIVE // use my IP
};
```

Which in my humble and very subjective opinion looks much cleaner. 

The second version uses a feature standardized in C99 called designated initializers. This allows initializing elements of an aggregate type in any order by specifying the array indices or structure field names. Elements that are not specified are initialized as if they are static objects: arithmetic types are initialized to `0`; pointers are initialized to `NULL`. This is also presumably less expensive than a `memset()` as it avoids clearing memory that will be assigned to.

If the structure has padding, using designated initializers will not initialize it to zero.  This is not *usually* a problem. However, this may inadvertently lead to information leaks when passing a structure from a more privileged context to a less privileged one: [Like when passing memory from the kernel back to userspace](https://lwn.net/Articles/417989/).

Now to be fair to Beej, he first wrote *The Guide* in 1995. But if you're writing new code in 2019, there is really no reason to not use designated initializers[^*]. Except to specifically avoid memory disclosures. *Cue the Rust Evangelism Strike Force chiming in to say that there is really no reason to be writing new C code in 2019.*

While we're on the topic of little annoyances, another pattern I often see is using a variable just to pass what is effectively a literal to `setsockopt()`:
```C
int yes=1;
setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof (yes)) ;
```

Which can instead be written as:
```C
setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &(int) {1}, sizeof(int));
```

This uses yet another C99 feature called compound literals. If the magic 1 value bothers you, you can always:
```C
setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &(int) {true}, sizeof(int));
```

And finally, there is absolutely no reason to check if a pointer is `NULL` just before calling `free()` on it:
```C
if(NULL != ptr)
	free(ptr);
``` 

Quoting the [standard Section 7.20.3.2 (p. 313)](http://www.open-std.org/JTC1/SC22/wg14/www/docs/n1124.pdf): *If ptr is a null pointer, no action occurs.* If a system deviates from this expectation, it is clearly evil. It is advised to *Kill It With Fire*.

[^*]:If you were relying on `memcmp()` to compare structs, that will probably no longer work. But you knew what you were getting yourself into, didn't you?
  