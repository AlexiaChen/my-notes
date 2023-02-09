# 程序员应该了解的操作系统各种IO知识

对于现在流行的什么大数据，分布式，消息队列等互联网系统，I/O是绕不过去的一个话题，之前也写过网络编程相关的IO话题 https://github.com/AlexiaChen/AlexiaChen.github.io/issues/79， 但是这次打算更加深入地讨论下。

## Buffered I/O 和 Direct I/O

一般开发者或许没有注意到这两者的区别，缓冲IO和直接IO是不太一样的：

- 缓冲IO： C语言的fopen fclose fwrite fread fflush fprintf fscanf等
- 直接IO： Linux的原生API，比如open close fsync read write pwrite pread等

缓冲IO实际上就是用户层还做了一个用户层的缓冲区，直接IO没有用户层缓冲区，但是操作系统内核的缓冲区还是有的。一般就是这两个不一样的地方。性能上有一定差距。

拿Linux来说，内核缓冲区其实也就是内核空间的一段buffer，只是这段buffer一般是以页为单位，Linux会把磁盘上的数据以页为单位缓存在操作系统内核中的内存里，一般来说，一个页是4K大小。

缓冲IO的读或写都要经过： 应用程序内存->用户缓冲区->内核缓冲区->磁盘， 其中的数据流转一次，从逻辑上看，有3次数据拷贝。内存中的数据拷贝大家都知道是很耗时的。高性能系统可能会压榨这里

直接IO的读或写都要经过： 应用程序内存->内核缓冲区->磁盘，有2次数据拷贝，性能相对要高。

所以你可以大概想到了，fflush只是把用户缓冲区的数据刷到内核缓冲区里，而fsync则是把内核缓冲区的数据刷到磁盘。

数据库的很多对事务的一致性和持久性底层实现确实依赖了fsync来“保证”数据落盘（fysnc等待落盘才返回，不信你可以搜LevelDB，SQLite，Redis等各种存储系统的源码，肯定能搜到fsync）。

但是从物理世界的角度看，fsync其实不是100$%可靠的，因为它只是把内核缓冲区的数据刷到磁盘的硬件缓冲区中，万一磁盘的磁头还没有把硬件缓冲区的数据写到磁盘上呢？这样数据也就丢失了。不过你可以暂时认为，fsync大部分情况是可靠的就可以了。你可以就认为，一旦fsync有返回，数据肯定落盘了，因为该函数是同步的。对于这种情况，就是耗性能，高性能和强一致性可能某种程度上是矛盾的，所以数据库的事务优化确实是很大一个研究领域。后面也会反复提到fsync这个函数。

## 内存映射文件与Zero Copy

相比于Direct IO，内存映射文件往前更进了一步，它是直接用应用程序的用户态内存映射到Linux内核缓冲区，应用程序读写自己的用户态内存相当于直接读写内核缓冲区。

内存映射文件的读写经过：应用程序内存（内核缓冲区） -> 磁盘，仅仅只有一次拷贝，性能更高。

熟悉Linux API就知道，mmap函数就是内存映射文件的API了，windows也有对应的API（CreateFileMapping， MapViewOfFile）。Java跨平台，封装了两种类型的API，提供一个更高层次的内存映射文件API叫MappedByteBuffer，底层原理都差不多。

Zero Copy又是提升IO性能的一个技术，Kafka在消费消息的时候就利用了Zero Copy技术。我们来看看最普通的，利用Direct IO实现的传输文件的网络发送代码:

```c
fd_file = opened file description
fd_socket = opened socket description
buffer = application bufffer
read(fd_file, buffer,...) // read data from local file, 2 times copy
write(fd_socket, buffer,...) // write data to socket, 2 times copy
```

以上代码总共发生了4次拷贝： 磁盘->内核缓冲区->应用程序内存buffer -> Socket buffer -> 网络

利用内存映射文件实现的同功能的代码:

```fd_file = opened file description
fd_socket = opened socket description
buffer = application bufffer
mmap(fd_file, buffer,...) // read data from local file, 1 times copy
write(fd_socket, buffer,...) // write data to socket, 2 times copyc
```

以上代码总共发生了3次拷贝： 磁盘 -> 应用程序内存buffer(内核缓冲区) -> Socket buffer -> 网络

但是，如果引用Zero Copy技术，内核缓冲区到Socket缓冲区的拷贝也可以省略了，也就是数据拷贝不会发生在用户态内存了。因为内核缓冲区与Socket buffer建立了映射（由DMA模块办到），没有用户态的事情了，直接减小用户态到内核态的开销切换，所以叫零拷贝。

过程是这样的: 磁盘 -> 内核缓冲区（Socket buffer） -> 网络

Linux上Zero Copy的API是sendfile，Java对应的是FileChannel.transferTo。

看API的名字就是知道，Zero Copy几乎是为实现网络上高性能传输文件而设计的。

还想性能再高？那请关注[DPDK](https://www.cnblogs.com/bakari/p/8404650.html),腾讯开源的F-stack就用了这种技术。

## 网络IO模型

之前写的服务端高性能网络编程的[文章](https://github.com/AlexiaChen/AlexiaChen.github.io/issues/79)有提到过。但是都特别简单，这次会稍微深入点。

之前的文章，总结起来就是，阻塞和非阻塞，函数是否立即返回。同步异步，读写是否由操作系统完成。下面讲点详细的。

- 同步阻塞IO

Linux的read，write函数，在调用的时候被阻塞，直到数据读写完成

- 同步非阻塞IO

也是read write函数，只是打开方式不同，打开文件描述符的时候带有O_NONBLOCK参数。于是，即使函数没有读写完成数据也会立即返回，不会阻塞，程序员需要编写代码主动去不断地轮询。

- IO多路复用

前面两种IO都只能用于简单的客户端开发，但是对于高性能服务端编程就需要处理很多的文件描述符。同步阻塞IO，每个线程只能处理一个文件描述符。同步非阻塞IO，要应用程序轮询大规模的文件描述符集合，不可行。所以就有了IO多路复用，一个线程能处理大规模的文件描述符集合。

Linux上有三种IO多路复用API：select，poll，epoll。其中epoll性能最高，主流都选择epoll。这三个API都是同步阻塞的。Java的NIO在Linux上就是基于poll或epoll的，有个selector来选择。

- 异步IO

目前只有windows下的IOCP完善成熟，Linux下也有aio，但是不成熟，不考虑。异步IO的接口几乎完全异步化。

所以当我们提及IO模型的时候，一般认为是操作系统层面的，但是谈及具体框架的IO模型，可能是框架封装出来的IO概念，例如boost.asio Java Netty。说白了，Java NIO就是poll epoll IOCP在JVM层面提供的一个语言级的统一接口，Netty框架又是基于NIO来做的。得看以后技术发展，Netty框架的异步IO概念底层实现可能是同步，也可能是真正的异步。


### Reactor和Proactor模式

请看：https://github.com/AlexiaChen/AlexiaChen.github.io/issues/79

上面讲的比较详细了。

### select，epoll的LT和ET

1. select

函数声明我就不具体写了，具体用法是，传入readfds和writefds这两个bit数组，函数返回的时候，readfds和writefds的bit位会发生变化，以此来告知那个文件描述符上有事件，需要遍历数组，然后调用相应的read或write。大致就那么个思想。

2. poll

跟select思想上差不多。不过多研究。

3. epoll

设计了一个逻辑上的epfd，epfd是个数字，把fd数组关联到上面，然后每次向内核传递的是epfd这个数字。

整个epoll过程分为以下三个步骤：

- 事件注册，通过epoll_ctl实现，对于服务端来说是accept，read，write三个事件
- 轮询这三个事件是否就绪，通过epoll_wait进行等待。一旦事件发生，该函数返回。
- 事件发生，执行实际IO操作，调用对应的IO函数accept，read，write

4. epoll的LT和ET模式

- LT（水平触发，条件触发），读缓冲区只要不为空，就会一直触发读事件；写缓冲区只要不满，就会一直触发写事件

- ET（边缘触发，状态触发），读缓冲区的状态从空转到非空的时候触发一次读事件；写缓冲区的状态从满转为非满的时候触发一次写事件。

从上面可以看到，ET模式实际做的事情应该会更少，并且一次到位，LT反而重复做了同样的事情，按理来说，ET性能会更高？但是实际上在工程中，大家一般用LT模式，这也是epoll的默认模式，Java的NIO用的也是LT模式。因为LT模式编写难度更高，容易遗漏事件，一次触发没处理好，不会给第二次机会了。

LT模式下因为写满缓冲区的概率很小，所以写完数据，记得取消注册的写事件。ET模式下，一旦读缓冲区有数据，记得一定把数据一次性读完，让缓冲区清空到空状态，不然就漏事件了。

根据陈硕说，目前没有任何实践证明ET性能比LT高，然后LT模式也更稳更安全些，所以大部分实践中都是默认LT模式了。

### 服务端编程的模型

一个监听线程，负责accept事件的注册和处理，和每个新进来的客户端建立socket连接，然后把socket连接交给IO线程，完成任务，继续监听新的客户端

N个IO线程，N一般等于CPU核数，负责每个socket连接上面read，write事件的注册和实际的socket的读写。把读到的Request放入Request队列，交由Workder线程异步处理

M个Worker线程，M的数量由具体业务决定，纯粹的业务线程，没有socket读写操作，对Request队列进行处理消费，生成Response队列，然后写入Response队列，交由IO线程处理，IO线程再回写数据给客户端。

大部分的服务端框架或服务器代码实现思路跟这个都差不多，大同小异。Tomcat6的NIO模块思路就是大致如此。










