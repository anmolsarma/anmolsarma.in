+++
title = "File Creation Time in Linux"
date = 2019-06-23T19:24:22+05:30
draft = false 
tags = ['tech', 'linux', 'filesystem']

+++

The [`stat`](http://man7.org/linux/man-pages/man1/stat.1.html) utility can be used to retrieve the Unix file timestamps namely `atime`, `ctime` and `mtime`. Of these, the benefit of `mtime` which records the last time when the file was modified is immediately apparent. On the other hand, `atime`[^1] which records the last time the file was accessed has been called ["perhaps the most stupid Unix design idea of all times"](https://lore.kernel.org/lkml/20070804210351.GA9784@elte.hu/). Intuitively, one might expect `ctime` to record the creation time of a file. However, `ctime` records the last time when the metadata of a file was changed.

Typically, Unices do not record file creation times. While some individual filesystems do record file creation times[^2], until recently Linux lacked a common interface to actually expose them to userspace applications. As a result, the output of `stat` (GNU coreutils v8.30) on an ext4 filesystem (Which does record creation times) looks something like this:

```bash
$ stat .
  File: .
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: 803h/2051d      Inode: 3588416     Links: 18
Access: (0775/drwxrwxr-x)  Uid: ( 1000/ anmol)   Gid: ( 1000/ anmol)
Access: 2019-06-23 10:49:04.056933574 +0000
Modify: 2019-05-19 13:29:59.609167627 +0000
Change: 2019-05-19 13:29:59.609167627 +0000
 Birth: -
```

With the "`Birth`" field, meant to show the creation time, sporting a depressing "`-`".

The fact that `ctime` does not mean creation time but change time coupled with the absence of a real creation time interface does lead to quite a bit of confusion. The confusion seems so pervasive that the `msdos` driver in the Linux kernel [happily clobbers](https://elixir.bootlin.com/linux/v5.1.14/source/fs/fat/inode.c#L883) the FAT creation time with the Unix change time!

The limitations of the current `stat()` system call have been known for some time. A new system call providing extended attributes was [first proposed in 2010](https://www.spinics.net/lists/linux-fsdevel/msg33831.html) with the new [`statx()`](https://lwn.net/Articles/685791/#statx) interface finally [being merged into Linux 4.11 in 2017](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a528d35e8bfcc521d7cb70aaf03e1bd296c8493f). It took so long at least in part because kernel developers quickly ran into one of the hardest problems in Computer Science: [naming things](https://lkml.org/lkml/2010/7/22/249). Because there was no standard to guide them, each filesystem took to calling creation time by a different name. [Ext4](https://elixir.bootlin.com/linux/v5.1.14/source/fs/ext4/ext4.h#L744) and [XFS](https://elixir.bootlin.com/linux/v5.1.14/source/fs/xfs/libxfs/xfs_inode_buf.h#L40) called it `crtime` while [Btrfs](https://elixir.bootlin.com/linux/v5.1.14/source/fs/btrfs/btrfs_inode.h#L187) and [JFS](https://elixir.bootlin.com/linux/v5.1.14/source/fs/jfs/jfs_incore.h#L46) called it `otime`. Implementations also have slightly different semantics with JFS storing creation time only with the [precision of seconds](https://elixir.bootlin.com/linux/v5.1.14/source/fs/jfs/jfs_imap.c#L3166).  

Glibc took a while to add a wrapper for statx() with support landing in [version 2.28](https://www.sourceware.org/ml/libc-alpha/2018-08/msg00003.html) which was released in 2018. Fast forward to March 2019 when GNU [coreutils 8.31](https://lists.gnu.org/archive/html/coreutils-announce/2019-03/msg00000.html) was released with `stat` finally gaining support for reading the file creation time:

```bash
$ stat .
  File: .
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: 803h/2051d      Inode: 3588416     Links: 18
Access: (0775/drwxrwxr-x)  Uid: ( 1000/ anmol)   Gid: ( 1000/ anmol)
Access: 2019-06-23 10:49:04.056933574 +0000
Modify: 2019-05-19 13:29:59.609167627 +0000
Change: 2019-05-19 13:29:59.609167627 +0000
 Birth: 2019-05-19 13:13:50.100925514 +0000
```

[^1]: The impact of `atime` on disk performance is mitigated by the use of [`relatime`](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/power_management_guide/relatime) on modern Linux systems.

[^2]: For ext4, one can get the `crtime` of a file using the `stat` subcommand of the confusingly named [`debugfs`](https://linux.die.net/man/8/debugfs) utility.