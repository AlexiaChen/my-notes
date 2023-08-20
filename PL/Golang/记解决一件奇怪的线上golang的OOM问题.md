为什么OOM打双引号，因为最终发现，不是内存泄露导致的OOM。

最近leader刚给我分配了一个OOM的任务给我，现象是explorer node在grafana的监控上出现在大量并发访问explorer node时，内存在监控上急剧上升，然后并没有下降，最终被systemd给kill掉了。

这种现象一般可以肯定都是OOM，就是代码哪里内存泄露了，导致分配的内存没有被GC。经过了我的一番排查，还写了一大段的英文分析报告，inuse_space是随着请求量增大而增高，但是随着请求量下降关闭，它也回归到了正常值，但是linux的top命令的RES项还是没有降低下来，所以当时我都怀疑人生了，是不是pprof哪里没用对或者哪里理解错了，后来还是请教了golang社区的一位哥们，他正好看过一个关于golang社区的提案，跟我说了，说的就是这个问题，很多人都遇到了，实际就是runtime GC掉的内存不会立刻归还给操作系统，所以就就给监控造成了麻烦。  这个feature主要是为了一些性能才这样做的，但是好心没办成好事。经过社区的讨论，最终还是恢复成原来的设置了，也就是在go 1.16中，被GC掉的内存会立刻归还给系统。

go的老版本需要` run binary with GODEBUG=madvdontneed=1` 就可以归还给系统了，或者直接升级到go 1.16。

一下是社区的一些讨论:

- https://github.com/golang/go/issues/42330
- https://github.com/golang/go/issues/37585
- https://go-review.googlesource.com/c/go/+/267100/
- https://github.com/google/cadvisor/issues/2242
- https://github.com/golang/go/issues/23687
- https://github.com/golang/go/issues/28466
- https://github.com/prometheus/prometheus/issues/5524

真的，如果不是那位哥们的提示，这么一个乌龙事件，我一时半会还查不出来，没看过那些提案一时半会也不好查，毕竟golang我是0项目经验，业余写的不算。虽然语言只是工具，但是毕竟坑，还是要花时间填的，这个就是经验。经验少就是没办法。

下面就是我写的分析报告:

[report.md](https://github.com/AlexiaChen/AlexiaChen.github.io/files/6403768/report.md)

其实这不光golang有这问题，一般系统自带的glibc的malloc都一般free了不会立刻归还给操作系统，如果是其他技术栈，可以考虑tcmalloc. 好奇可以看看这位知乎上的老哥也遇到了同样的问题，只不过不是golang而已，据说已经是被glibc malloc坑了不知道第几个案例了 
- https://www.zhihu.com/question/34787444/answer/2251415537  
- https://www.zhihu.com/question/53627438/answer/2247970485
- https://www.zhihu.com/question/310052411/answer/2227778788

