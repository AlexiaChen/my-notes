名字叫做`fiu-run`

可以模拟IO的失败概率，复现一些难复现的bug，混沌工程里面经常用。

主要之前研究分布式系统是怎么测试发现的。感谢PingCAP的文章。

当然，其他的相关工具还很多，包括模拟网卡丢包，乱序等，还有内存分配失败。但是这个最简单，其他的工具搭建环境有些麻烦。

已经对我目前所在的项目测试出一些不太stable的问题了。

参考：

-   [https://blitiri.com.ar/p/libfiu/doc/man-fiu-run.html](https://blitiri.com.ar/p/libfiu/doc/man-fiu-run.html)
-   [https://pingcap.com/blog-cn/distributed-system-test-2/](https://pingcap.com/blog-cn/distributed-system-test-2/)