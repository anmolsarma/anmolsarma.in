+++
title = "Stop Memsetting Structures"
date = 2019-04-27T23:59:42+05:30
draft = false 
tags = ['tech', 'C', 'rant', 'language-lawyering']

+++


*TL;DR: Use C99's designated initializers instead. Because it's 2019!*

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
	.ai_family = AF_UNSPEC;
	.ai_socktype = SOCK_STREAM;
	.ai_flags = AI_PASSIVE; // use my IP
};
```

Which in my humble and very subjective opinion looks much cleaner. 

The second version uses a feature standardized in C99 called designated initializers. This allows initializing elements of an aggregate type in any order by specifying the array indices or structure field names. Elements that are not specified are initialized as if they are static objects: arithmetic types are initialized to `0`; pointers are initialized to `NULL`.

Now to be fair to Beej, he first wrote *The Guide* in 1995. But if you're writing new code in 2019, there is really no reason to not use designated initializers. *Cue the Rust Evangelism Strike Force chiming in to say that there is really no reason to be writing new C code in 2019.* 

While we're on the topic of little annoyances, another pattern I often see is using a variable just to pass what is effectively a literal to `setsockopt()`:
```C
int yes=1;
setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof (yes)) ;
```

Which can instead be written as:
```C
setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &(int) {1}, sizeof(int));
```

And finally, there is absolutely no reason to check if a pointer is `NULL` just before calling `free()` on it:
```C
if(NULL != ptr)
	free(ptr);
``` 

Quoting the [standard](http://www.open-std.org/JTC1/SC22/wg14/www/docs/n1124.pdf): *If ptr is a null pointer, no action occurs.*
  