+++
title = "Network Redirections in Bash"
date = 2019-05-04T20:59:15+05:30
draft = false
tags = ['tech', 'linux']

+++

A few months ago, while reading the man page for `recvmmsg()`, I came across this snippet:
```bash
$ while true; do echo $RANDOM > /dev/udp/127.0.0.1/1234;
     sleep 0.25; done
```

And as advertised, it sends a UDP datagram containing a random number to port 1234 every 250 ms. I didn't recall ever seeing a `/dev/udp` and so was a bit surprised that it worked. And as it happens, `ls` was not able to access the file that I had just written to:
```bash 
ls: cannot access '/dev/udp/127.0.0.1/1234': No such file or directory
```

Puzzled and intrigued, I `echoe`d *Foo Bar Baz* to `/dev/udp/127.0.0.1/1337` and reached for `strace`:
```C
...
2423 socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP) = 4
12423 connect(4, {sa_family=AF_INET, sin_port=htons(1337), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
12423 fcntl(1, F_GETFD)                 = 0
12423 fcntl(1, F_DUPFD, 10)             = 10
12423 fcntl(1, F_GETFD)                 = 0
12423 fcntl(10, F_SETFD, FD_CLOEXEC)    = 0
12423 dup2(4, 1)                        = 1
12423 close(4)                          = 0
12423 fstat(1, {st_mode=S_IFSOCK|0777, st_size=0, ...}) = 0
12423 write(1, "Foo Bar Baz\n", 12)     = 12
...
```

Seemingly, a normal UDP socket was being created and written to using the regular sycall interface. That refuted my initial suspicion that some kind of a special file backed by the kernel was involved. But who was actually creating the socket?

A peek at Bash's code answered that question:

**redir.c:**
```C
/* A list of pattern/value pairs for filenames that the redirection
   code handles specially. */
static STRING_INT_ALIST _redir_special_filenames[] = {
#if !defined (HAVE_DEV_FD)
  { "/dev/fd/[0-9]*", RF_DEVFD },
#endif
#if !defined (HAVE_DEV_STDIN)
  { "/dev/stderr", RF_DEVSTDERR },
  { "/dev/stdin", RF_DEVSTDIN },
  { "/dev/stdout", RF_DEVSTDOUT },
#endif
#if defined (NETWORK_REDIRECTIONS)
  { "/dev/tcp/*/*", RF_DEVTCP },
  { "/dev/udp/*/*", RF_DEVUDP },
#endif
  { (char *)NULL, -1 }
};
```


So, redirection involving `/dev/udp/` is handled specially by Bash[^1] and it uses BSD Sockets API to create a socket:

[^1]: At least on Linux, the other special patterns handled by bash like [`/dev/fd`](http://www.informit.com/articles/article.aspx?p=99706&seqNum=15) and `/dev/stdint` actually are special files backed by the kernel. The [Bash manual](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) notes that it may emulate them internally on platforms that do not support them. 

**lib/sh/netopen.c:**
```C
/*
 * Open a TCP or UDP connection to HOST on port SERV.  Uses the
 * traditional BSD mechanisms.  Returns the connected socket or -1 on error.
 */
static int
_netopen4(host, serv, typ)
     char *host, *serv;
     int typ;
{
  struct in_addr ina;
  struct sockaddr_in sin;
  unsigned short p;
  int s, e;

  if (_getaddr(host, &ina) == 0)
    {
      internal_error (_("%s: host unknown"), host);
      errno = EINVAL;
      return -1;
    }

  if (_getserv(serv, typ, &p) == 0)
    {
      internal_error(_("%s: invalid service"), serv);
      errno = EINVAL;
      return -1;
    }

  memset ((char *)&sin, 0, sizeof(sin));
  sin.sin_family = AF_INET;
  sin.sin_port = p;
  sin.sin_addr = ina;

  s = socket(AF_INET, (typ == 't') ? SOCK_STREAM : SOCK_DGRAM, 0);
  if (s < 0)
    {
      sys_error ("socket");
      return (-1);
    }

  if (connect (s, (struct sockaddr *)&sin, sizeof (sin)) < 0)
    {
      e = errno;
      sys_error("connect");
      close(s);
      errno = e;
      return (-1);
    }

  return(s);
}
```

Which means we can actually make HTTP requests using Bash:
```bash
exec 3<> /dev/tcp/checkip.amazonaws.com/80
printf "GET / HTTP/1.1\r\nHost: checkip.amazonaws.com\r\nConnection: close\r\n\r\n" >&3
tail -n1 <&3
```
No `curl` needed! /jk

Apart from Bash, in the versions and configurations packaged in Ubuntu 18.04, only `ksh` supports network redirections -- `ash`, `csh`, `dash`,  `fish`,  and `zsh` do not.

I don't think I will actually have any use for network redirections but this was a fun little rabbit hole to dive into.

**NOTE:** Code snippets from Bash are licensed under GPLv3, the snippet from the man page is licensed [differently](http://man7.org/linux/man-pages/man2/recvmmsg.2.license.html)
 
 
