# 深入解析fsync()

## 前言

整理这篇文章最开始的起因是室友它想做一些性能方面的测试，需要把磁盘缓冲区清空了，为此我就摸索查了些相关资料。刚好在SO上找到了这个 https://stackoverflow.com/questions/9551838/how-to-purge-disk-i-o-caches-on-linux， 与室友的场景需求很一致。


可能大家看过数据库存储引擎相关源码的会对这个POSIX标准的API有了解(fsync fdatasync等)，因为需要它来配合实现数据本地事务的ACID中的D。调用它可以确保数据被写入物理硬盘，而这么一个简单的API描述背后，却有复杂的实现，与文件系统也相关，而且有些时候，并不保证落盘。

于此在业内发生了很多大事，就比如PostgreSQL的数据库团队错误使用fysnc有20年之久，hacker news上报道，标题为《PostgreSQL used fysnc incorrectly for 20 years》。连这些专家都会犯错，所以我们对这些看似简单的事物还是要保持敬畏和好奇的，正因为这样，这fsync()还是值得深究一番的，为此整理这个脉络。

## 正文

首先来看，一些关于fsync()的变化。第一先看[POSIX标准](https://pubs.opengroup.org/onlinepubs/009695399/functions/fsync.html)：

```txt
The fsync() function shall request that all data for the open file descriptor named by fildes is to be transferred to the storage device associated with the file described by fildes. The nature of the transfer is implementation-defined. The fsync() function shall not return until the system has completed that action or until an error is detected.

[SIO] [Option Start] If _POSIX_SYNCHRONIZED_IO is defined, the fsync() function shall force all currently queued I/O operations associated with the file indicated by file descriptor fildes to the synchronized I/O completion state. All I/O operations shall be completed as defined for synchronized I/O file integrity completion. 


Relatinal:

    The fsync() function is intended to force a physical write of data from the buffer cache, and to assure that after a system crash or other failure that all data up to the time of the fsync() call is recorded on the disk. Since the concepts of "buffer cache", "system crash", "physical write", and "non-volatile storage" are not defined here, the wording has to be more abstract.

    If _POSIX_SYNCHRONIZED_IO is not defined, the wording relies heavily on the conformance document to tell the user what can be expected from the system. It is explicitly intended that a null implementation is permitted. This could be valid in the case where the system cannot assure non-volatile storage under any circumstances or when the system is highly fault-tolerant and the functionality is not required. In the middle ground between these extremes, fsync() might or might not actually cause data to be written where it is safe from a power failure. The conformance document should identify at least that one configuration exists (and how to obtain that configuration) where this can be assured for at least some files that the user can select to use for critical data. It is not intended that an exhaustive list is required, but rather sufficient information is provided so that if critical data needs to be saved, the user can determine how the system is to be configured to allow the data to be written to non-volatile storage.

    It is reasonable to assert that the key aspects of fsync() are unreasonable to test in a test suite. That does not make the function any less valuable, just more difficult to test. A formal conformance test should probably force a system crash (power shutdown) during the test for this condition, but it needs to be done in such a way that automated testing does not require this to be done except when a formal record of the results is being made. It would also not be unreasonable to omit testing for fsync(), allowing it to be treated as a quality-of-implementation issue.

```

由上面可以看到，标准说是这么说了，但是有些细节行为是implementation-defined的，也就是还是要看实现。实现方面我查了些资料，得出的结论是：

对于Linux下的ext3文件系统来说，如果底层的物理硬盘写缓冲是打开的，那么即使fysnc或fdatasync函数返回，数据也不会被持久化到物理硬盘。

而对于Linux的ext4文件系统来说，如果是在比较老的内核上，亦或者其他使用较少的文件系统上，那也是不确定行为，因为不知道它具体是怎么刷磁盘缓存的（如果磁盘缓存被打开），这些情况，你要手动关闭磁盘缓存，关闭磁盘缓存可以使用 hdparm(8) sdparm(8) 这两个命令来做。关于这些，这里有个[issue提案](https://lwn.net/Articles/270891/)。

当然，对于现在的Linux内核，早就加入了[写屏障](https://docs.fedoraproject.org/en-US/Fedora/14/html/Storage_Administration_Guide/writebarr.html)了,所以无需担心。

举个例子，是老版本的ubuntu和较新的ubuntu的fysnc man pages文档描述对比:

ubuntu 8.04(hardy):

```txt
DESCRIPTION

fsync() transfers (“flushes”) all modified in-core data of (i.e., modified buffer cache pages for) the file referred to by the file descriptor fd to the disk device (or other permanent storage device) where that file resides. The call blocks until the device reports that the transfer has completed. It also flushes metadata information associated with the file (see stat(2)).

Calling fsync() does not necessarily ensure that the entry in the directory containing the file has also reached disk. For that an explicit fsync() on a file descriptor for the directory is also needed.

NOTES

Applications that access databases or log files often write a tiny data fragment (e.g., one line in a log file) and then call fsync() immediately in order to ensure that the written data is physically stored on the harddisk. Unfortunately, fsync() will always initiate two write operations: one for the newly written data and another one in order to update the modification time stored in the inode. If the modification time is not a part of the transaction concept fdatasync() can be used to avoid unnecessary inode disk write operations.

If the underlying hard disk has write caching enabled, then the data may not really be on permanent storage when fsync() / fdatasync() return.

When an ext2 file system is mounted with the sync option, directory entries are also implicitly synced by fsync().

On kernels before 2.4, fsync() on big files can be inefficient. An alternative might be to use the O_SYNC flag to open(2).

In Linux 2.2 and earlier, fdatasync() is equivalent to fsync(), and so has no performance advantage.
```

ubuntu 14.04:

```txt
DESCRIPTION

fsync() transfers (“flushes”) all modified in-core data of (i.e., modified buffer cache pages for) the file referred to by the file descriptor fd to the disk device (or other permanent storage device) so that all changed information can be retrieved even after the system crashed or was rebooted. This includes writing through or flushing a disk cache if present. The call blocks until the device reports that the transfer has completed. It also flushes metadata information associated with the file (see stat(2)).

Calling fsync() does not necessarily ensure that the entry in the directory containing the file has also reached disk. For that an explicit fsync() on a file descriptor for the directory is also needed.

NOTES

On some UNIX systems (but not Linux), fd must be a writable file descriptor.

In Linux 2.2 and earlier, fdatasync() is equivalent to fsync(), and so has no performance advantage.

The fsync() implementations in older kernels and lesser used filesystems does not know how to flush disk caches. In these cases disk caches need to be disabled using hdparm(8) or sdparm(8) to guarantee safe operation.
```

嗯，看描述，看来新版的Linux几乎不用担心了，只要fsync正常返回，那么就保证数据写入物理磁盘，内核崩溃，断电都不怕了，数据肯定被持久化到硬盘了，然而，2018年发生了一个大事，就是之前提到的《PostgreSQL used fysnc incorrectly for 20 years》事件，PostgreSQL用错了这API，当然包括MySQL的InnoDB，MongoDB的WiredTiger存储引擎肯定也错了，因为数据库存储引擎团队圈子小，他们都用错了！

最后PostgreSQL 12的时候修复了这个bug，commit[在这里](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ccdd7f66e3324d2b6d3dec282cfa9ff084083f1)，然后也把这个commit的改变修复了以前的PostgreSQL版本。

为什么修复这个bug呢？commit的log也解释了：

```txt
PANIC on fsync() failure.

On some operating systems, it doesn't make sense to retry fsync(),
because dirty data cached by the kernel may have been dropped on
write-back failure.  In that case the only remaining copy of the
data is in the WAL.  A subsequent fsync() could appear to succeed,
but not have flushed the data.  That means that a future checkpoint
could apparently complete successfully but have lost data.

Therefore, violently prevent any future checkpoint attempts by
panicking on the first fsync() failure.  Note that we already
did the same for WAL data; this change extends that behavior to
non-temporary data files.

Provide a GUC data_sync_retry to control this new behavior, for
users of operating systems that don't eject dirty data, and possibly
forensic/testing uses.  If it is set to on and the write-back error
was transient, a later checkpoint might genuinely succeed (on a
system that does not throw away buffers on failure); if the error is
permanent, later checkpoints will continue to fail.  The GUC defaults
to off, meaning that we panic.

Back-patch to all supported releases.

There is still a narrow window for error-loss on some operating
systems: if the file is closed and later reopened and a write-back
error occurs in the intervening time, but the inode has the bad
luck to be evicted due to memory pressure before we reopen, we could
miss the error.  A later patch will address that with a scheme
for keeping files with dirty data open at all times, but we judge
that to be too complicated to back-patch.

```

好了，现在我解释下，fsync失败出错了嘛，按照正常的程序逻辑，出错就再次重试fsync呗，直到成功为止，这个逻辑没错吧，但是事实不是这样的，fsync函数出现EIO错误失败了，不能重试，因为你重试就是肯定正常返回了，**因为这个EIO错误只会出现一次！** 这样做的话，就导致重试直到正常返回了，你以为数据肯定落盘了，但是实际没有，那糟糕了，那事务的ACID就可能在这样的边界条件下得不到保证，那么就是错的。所以，他们给出的修复方法是，**一旦出现EIO错误，就立即abort程序，这个是硬件错误，不能再继续下去了**。

InnoDB和WiredTiger存储引擎也对这种错误用法给了修复，以下是两个commit:

- https://github.com/mysql/mysql-server/commit/8590c8e12a3374eeccb547359750a9d2a128fa6a#diff-28ed40e7ccec6683ebd46da2ca82d01c

- https://github.com/wiredtiger/wiredtiger/commit/ae8bccce3d8a8248afa0e4e0cf67674a43dede96


## 参考

- http://blog.httrack.com/blog/2013/11/15/everything-you-always-wanted-to-know-about-fsync/
- https://wiki.postgresql.org/wiki/Fsync_Errors
- https://lwn.net/Articles/752063/
- https://stackoverflow.com/questions/9551838/how-to-purge-disk-i-o-caches-on-linux
