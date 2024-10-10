#flashcards 
## 为什么顺序IO比随机IO快?::
随机IO要进行磁头的移动，而顺序IO依靠的旋转，磁盘转速肯定比磁头移动要快！

## 操作系统每次从磁盘读取多少数据? ::
4KB

## IO模型

### Unix的五种IO模型
#### 阻塞IO

![](阻塞IO.png)

缺点：
1. 系统调用阻塞
2. 一个请求对应一个线程
3. 并发数较大时，需要创建大量线程，系统资源占用大

#### 非阻塞IO

![](非阻塞IO.png)

优点：解决了BIO系统调用阻塞的问题

缺点：用户程序需要遍历socket集合，O（n）次的系统调用

#### 多路复用IO

![](多路复用IO.png)

优点：一次select、poll、epoll等系统调用，由内核进行遍历文件描述符，返回产生事件的文件描述符（socket）

#### 信号驱动IO

![](信号驱动IO.png)

#### 异步IO

![](异步IO.png)
#### 文章引用
https://zhuanlan.zhihu.com/p/115912936
### select、poll、epoll
select、poll、epoll是**多路复用IO**的具体实现方式。
#### select

##### 原理

![](select.jpg)

1. 使用`copy_from_user`从用户空间拷贝`fd_set`到内核空间
2. 注册回调函数`__pollwait`
3. 遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）
4. 以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。
5. __pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。
6. poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。
7. 如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。
8. 把`fd_set`从内核空间拷贝到用户空间。

##### 缺点

1. 每次调用select，都需要把`fd_set`集合从用户态拷贝到内核态，如果`fd_set`集合很大时，那这个开销也很大
2. 同时每次调用select都需要在内核遍历传递进来的所有`fd_set`，如果`fd_set`集合很大时，那这个开销也很大
    > 对socket是线性扫描，即轮询，效率较低： 仅知道有I/O事件发生，却不知是哪几个流，只会无差异轮询所有流，找出能读数据或写数据的流进行操作。同时处理的流越多，无差别轮询时间越长 - O(n)
3. 为了减少数据拷贝带来的性能损坏，内核对被监控的`fd_set`集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)
    > 一般该数和系统内存关系很大，具体数目可以`cat /proc/sys/fs/file-max`查看。32位机默认1024个，64位默认2048。现代操作系统这个数可能更大。

#### **poll**

poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，其他的都差不多，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll没有最大文件描述符数量的限制。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

- 它将用户传入的数组拷贝到内核空间
- 然后查询每个fd对应的设备状态：
- 如果设备就绪 在设备等待队列中加入一项继续遍历
- 若遍历完所有fd后，都没发现就绪的设备，挂起当前进程，直到设备就绪或主动超时，被唤醒后它又再次遍历fd。这个过程经历多次无意义的遍历。

相比起select，poll实现方式没有最大连接数限制，因其基于链表存储。

##### 缺点
1. 大量fd数组被整体复制于用户态和内核地址空间间，而不管是否有意义
2. 如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd

#### epoll

可理解为**event poll**，epoll会把哪个流发生哪种I/O事件通知我们。所以epoll是事件驱动（每个事件关联fd）的，此时我们对这些流的操作都是有意义的。复杂度也降低到了O(1)。

##### 触发模式

- **EPOLLLT**
    - LT，默认的模式（水平触发） 只要该fd还有数据可读，每次 `epoll_wait` 都会返回它的事件，提醒用户程序去操作
- **EPOLLET**
    - ET是“高速”模式（边缘触发），只会提示一次，直到下次再有数据流入之前都不会再提示，无论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读完，即读到read返回值小于请求值或遇到EAGAIN错误

##### EPOLLET触发模式的意义

若用`EPOLLLT`，系统中一旦有大量无需读写的就绪文件描述符，它们每次调用`epoll_wait`都会返回，这大大降低处理程序检索自己关心的就绪文件描述符的效率。 而采用`EPOLLET`，当被监控的文件描述符上有可读写事件发生时，`epoll_wait`会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用`epoll_wait`时，它不会通知你，即只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。这比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。

##### 优点

- 没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）
- 效率提升，不是轮询，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数 即Epoll最大的优点就在于它只关心“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll
- 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。
- epoll通过内核和用户空间共享一块内存来实现的

**表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调**。

epoll跟select都能提供多路I/O复用的解决方案。在现在的Linux内核里有都能够支持，其中epoll是Linux所特有，而select则应该是POSIX所规定，一般操作系统均有实现。

select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，`epoll_create`,`epoll_ctl`和`epoll_wait`，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。

###### 与select和poll对比

1. 对于select和poll每次调用都拷贝fd的缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。
2. epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。
3. 对于select文件描述符管理上限的缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

##### 原理

* epoll_create：创建一棵红黑树，返回一个文件描述符
* epoll_ctl：对文件描述符关联的红黑树进行增删改
* epoll_wait：检测epoll树上就绪状态的文件描述符，如果没有会阻塞，可以设置超时时间

##### 参考文章
[如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （1） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/63179839)
[如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （2） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/64138532)
[如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （3） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/64746509)

#### 总结

select，poll，epoll都是IO多路复用机制，即可以监视多个描述符，一旦某个描述符就绪（读或写就绪），能够通知程序进行相应读写操作。 但select，poll，epoll本质上都是**同步I/O**，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

- select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。
- select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。


## Reactor

Reactor模式，属于I/O多路复用的一种常见模式。 IO多路复用( I/O multiplexing )指的通过单个线程管理多个Socket。

Reactor pattern(反应器设计模式)是一种为处理并发服务请求，并将请求提交到一个或者多个服务处理程序的事件设计模式。

Reactor模式是事件驱动的有一个或多个并发输入源(文件事件)。有一个Service Handler，有多个Request Handlers。

这个Service Handler会同步的将输入的请求(Event)多路复用的分发给相应的Request Handler

![](https://secure2.wostatic.cn/static/q3svttSYJV8mbnfu9suqkv/image.png?auth_key=1716829338-2N7HtHFTNR6joSBkRiVNR5-0-f4f77fee27dc55e9e923f8ad839efd6d)

![](https://secure2.wostatic.cn/static/qcKmW5eD16mvnMr6mpYQE9/image.png?auth_key=1716829339-iYsGxoCrBtMPV6yyTNSzrS-0-912bc7df82c1d3341ad1d6f20a5e5ac4)

- Handle：I/O操作的基本文件句柄，在linux下就是fd(文件描述符)
- Synchronous Event Demultiplexer：同步事件分离器，阻塞等待Handles中的事件发生。(系统)
- Reactor：事件分派器，负责事件的注册，删除以及对所有注册到事件分派器的事件进行监控， 当事件发生时会调用Event Handler接口来处理事件。
- Event Handler： 事件处理器接口，这里需要Concrete Event Handler来实现该接口
- Concrete Event Handler：真实的事件处理器，通常都是绑定了一个handle，实现对可读事件 进行读 取或对可写事件进行写入的操作。

![](https://secure2.wostatic.cn/static/63SD6imMvRiaf6tqNBiHsK/image.png?auth_key=1716829338-ua7QWVWKoxL4K4BxXVEnAs-0-d336414f5c4ad6b910156d12f16a07cd)

1. 主程序向事件分派器(Reactor)注册要监听的事件
2. Reactor调用OS提供的事件处理分离器，监听事件(wait)
3. 当有事件产生时，Reactor将事件派给相应的处理器来处理 handle_event()

4种IO多路复用模型与选择 `select`，`poll`，`epoll`、`kqueue`都是IO多路复用的机制。

I/O多路复用就是通过一种机制，一个进程可以监视多个描述符(socket)，一旦某个描述符就绪(一 般是读就绪或者写就绪)，能够通知程序进行相应的读写操作。

select、poll、epoll
## 零拷贝::
### 什么是零拷贝？::
零拷贝（Zero-copy）技术指在计算机执行操作时，CPU 不需要先将数据从一个内存区域复制到另一个内存区域，从而可以减少上下文切换以及 CPU 的拷贝时间。

- 零拷贝机制可以减少数据在内核缓冲区和用户进程缓冲区之间反复的 I/O 拷贝操作。
- 零拷贝机制可以减少用户进程地址空间和内核地址空间之间因为上下文切换而带来的 CPU 开销。
### 传统的拷贝
![](传统拷贝.webp)
### 零拷贝有哪些方式？::
在 Linux 中零拷贝技术主要有 3 个实现思路：**用户态直接 I/O**、**减少数据拷贝次数**以及**写时复制技术**。

- 用户态直接 I/O：应用程序可以直接访问硬件存储，操作系统内核只是辅助数据传输。这种方式依旧存在用户空间和内核空间的上下文切换，硬件上的数据直接拷贝至了用户空间，不经过内核空间。因此，直接 I/O 不存在内核空间缓冲区和用户空间缓冲区之间的数据拷贝。
- 减少数据拷贝次数：在数据传输过程中，避免数据在用户空间缓冲区和系统内核空间缓冲区之间的CPU拷贝，以及数据在系统内核空间内的CPU拷贝，这也是当前主流零拷贝技术的实现思路。
	- **mmap + write**
	- **sendfile**
	- **sendfile + DMA gather copy**
	- **splice**
- 写时复制技术：写时复制指的是当多个进程共享同一块数据时，如果其中一个进程需要对这份数据进行修改，那么将其拷贝到自己的进程地址空间中，如果只是数据读取操作则不需要进行拷贝操作。
- 缓冲区共享：缓冲区共享方式完全改写了传统的 I/O 操作，因为传统 I/O 接口都是基于数据拷贝进行的，要避免拷贝就得去掉原先的那套接口并重新改写，所以这种方法是比较全面的零拷贝技术，目前比较成熟的一个方案是在 Solaris 上实现的 fbuf（Fast Buffer，快速缓冲区）。
#### mmap + write
一种零拷贝方式是使用 mmap + write 代替原来的 read + write 方式，减少了 1 次 CPU 拷贝操作。mmap 是 Linux 提供的一种内存映射文件方法，即将一个进程的地址空间中的一段虚拟地址映射到磁盘文件地址，mmap + write 的伪代码如下：

```cpp
tmp_buf = mmap(file_fd, len);
write(socket_fd, tmp_buf, len);
```

使用 mmap 的目的是将内核中读缓冲区（read buffer）的地址与用户空间的缓冲区（user buffer）进行映射，从而实现内核缓冲区与应用程序内存的共享，省去了将数据从内核读缓冲区（read buffer）拷贝到用户缓冲区（user buffer）的过程，然而内核读缓冲区（read buffer）仍需将数据到内核写缓冲区（socket buffer），大致的流程如下图所示：

![](https://pic2.zhimg.com/v2-28463616753963ac9f189ce23a485e2d_b.jpg)

![](mmap.webp)
基于 mmap + write 系统调用的零拷贝方式，整个拷贝过程会发生 4 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝，用户程序读写数据的流程如下：

1. 用户进程通过 mmap() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2. 将用户进程的内核空间的读缓冲区（read buffer）与用户空间的缓存区（user buffer）进行内存地址映射。
3. CPU利用DMA控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
4. 上下文从内核态（kernel space）切换回用户态（user space），mmap 系统调用执行返回。
5. 用户进程通过 write() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
6. CPU将读缓冲区（read buffer）中的数据拷贝到的网络缓冲区（socket buffer）。
7. CPU利用DMA控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
8. 上下文从内核态（kernel space）切换回用户态（user space），write 系统调用执行返回。

mmap 主要的用处是提高 I/O 性能，特别是针对大文件。对于小文件，内存映射文件反而会导致碎片空间的浪费，因为内存映射总是要对齐页边界，最小单位是 4 KB，一个 5 KB 的文件将会映射占用 8 KB 内存，也就会浪费 3 KB 内存。

mmap 的拷贝虽然减少了 1 次拷贝，提升了效率，但也存在一些隐藏的问题。当 mmap 一个文件时，如果这个文件被另一个进程所截获，那么 write 系统调用会因为访问非法地址被 SIGBUS 信号终止，SIGBUS 默认会杀死进程并产生一个 coredump，服务器可能因此被终止。
#### sendfile
sendfile 系统调用在 Linux 内核版本 2.1 中被引入，目的是简化通过网络在两个通道之间进行的数据传输过程。sendfile 系统调用的引入，不仅减少了 CPU 拷贝的次数，还减少了上下文切换的次数，它的伪代码如下：

```cpp
sendfile(socket_fd, file_fd, len);
```

通过 sendfile 系统调用，数据可以直接在内核空间内部进行 I/O 传输，从而省去了数据在用户空间和内核空间之间的来回拷贝。与 mmap 内存映射方式不同的是， sendfile 调用中 I/O 数据对用户空间是完全不可见的。也就是说，这是一次完全意义上的数据传输过程。

![](https://pic3.zhimg.com/v2-48132735369375701f3d8ac1d6029c2a_b.jpg)

![](sendfile.webp)
基于 sendfile 系统调用的零拷贝方式，整个拷贝过程会发生 2 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝，用户程序读写数据的流程如下：

1. 用户进程通过 sendfile() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2. CPU 利用 DMA 控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
3. CPU 将读缓冲区（read buffer）中的数据拷贝到的网络缓冲区（socket buffer）。
4. CPU 利用 DMA 控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
5. 上下文从内核态（kernel space）切换回用户态（user space），sendfile 系统调用执行返回。

相比较于 mmap 内存映射的方式，sendfile 少了 2 次上下文切换，但是仍然有 1 次 CPU 拷贝操作。sendfile 存在的问题是用户程序不能对数据进行修改，而只是单纯地完成了一次数据传输过程。
#### sendfile + DMA gather copy
Linux 2.4 版本的内核对 sendfile 系统调用进行修改，为 DMA 拷贝引入了 gather 操作。它将内核空间（kernel space）的读缓冲区（read buffer）中对应的数据描述信息（内存地址、地址偏移量）记录到相应的网络缓冲区（ socket buffer）中，由 DMA 根据内存地址、地址偏移量将数据批量地从读缓冲区（read buffer）拷贝到网卡设备中，这样就省去了内核空间中仅剩的 1 次 CPU 拷贝操作，sendfile 的伪代码如下：

```cpp
sendfile(socket_fd, file_fd, len);
```

在硬件的支持下，sendfile 拷贝方式不再从内核缓冲区的数据拷贝到 socket 缓冲区，取而代之的仅仅是缓冲区文件描述符和数据长度的拷贝，这样 DMA 引擎直接利用 gather 操作将页缓存中数据打包发送到网络中即可，本质就是和虚拟内存映射的思路类似。

![](https://pic4.zhimg.com/v2-15edf2971101883e2a90253225a3b0d3_b.jpg)

![](sendfile-gather.webp)
基于 sendfile + DMA gather copy 系统调用的零拷贝方式，整个拷贝过程会发生 2 次上下文切换、0 次 CPU 拷贝以及 2 次 DMA 拷贝，用户程序读写数据的流程如下：

1. 用户进程通过 sendfile() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2. CPU 利用 DMA 控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
3. CPU 把读缓冲区（read buffer）的文件描述符（file descriptor）和数据长度拷贝到网络缓冲区（socket buffer）。
4. 基于已拷贝的文件描述符（file descriptor）和数据长度，CPU 利用 DMA 控制器的 gather/scatter 操作直接批量地将数据从内核的读缓冲区（read buffer）拷贝到网卡进行数据传输。
5. 上下文从内核态（kernel space）切换回用户态（user space），sendfile 系统调用执行返回。

sendfile + DMA gather copy 拷贝方式同样存在用户程序不能对数据进行修改的问题，而且本身需要硬件的支持，它只适用于将数据从文件拷贝到 socket 套接字上的传输过程。
#### splice
sendfile 只适用于将数据从文件拷贝到 socket 套接字上，同时需要硬件的支持，这也限定了它的使用范围。Linux 在 2.6.17 版本引入 splice 系统调用，不仅不需要硬件支持，还实现了两个文件描述符之间的数据零拷贝。splice 的伪代码如下：

```cpp
splice(fd_in, off_in, fd_out, off_out, len, flags);
```

splice 系统调用可以在内核空间的读缓冲区（read buffer）和网络缓冲区（socket buffer）之间建立管道（pipeline），从而避免了两者之间的 CPU 拷贝操作。

![](https://pic2.zhimg.com/v2-37cf7a8b129183c24c7b524d3fee1a29_b.jpg)

基于 splice 系统调用的零拷贝方式，整个拷贝过程会发生 2 次上下文切换，0 次 CPU 拷贝以及 2 次 DMA 拷贝，用户程序读写数据的流程如下：

1. 用户进程通过 splice() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2. CPU 利用 DMA 控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
3. CPU 在内核空间的读缓冲区（read buffer）和网络缓冲区（socket buffer）之间建立管道（pipeline）。
4. CPU 利用 DMA 控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
5. 上下文从内核态（kernel space）切换回用户态（user space），splice 系统调用执行返回。

splice 拷贝方式也同样存在用户程序不能对数据进行修改的问题。除此之外，它使用了 Linux 的管道缓冲机制，可以用于任意两个文件描述符中传输数据，但是它的两个文件描述符参数中有一个必须是管道设备。
###  Linux零拷贝对比

| 拷贝方式 | CPU拷贝 | DMA拷贝 | 系统调用 | 上下文切换 |
| ------  | ------ | ------ | ------ | ------ |
| 传统方式（read + write） | 2 | 2 | read/write | 4 |
| 内存映射（mmap + write） | 1 | 2 | mmap/write| 4|
| sendfile | 1 | 2 |sendfile | 2 |
| sendfile + DMA gather copy | 0 | 2 |sendfile | 2 |
| splice     | 0   | 2 |splice   | 2 |
### 零拷贝有哪些应用？::
* Java NIO
	* MappedByteBuffer：mmap
	* DirectByteBuffer：mmap
	*  FileChannel：sendfile
* Netty零拷贝
	* Netty 通过 DefaultFileRegion 类对 java.nio.channels.FileChannel 的 tranferTo() 方法进行包装，在文件传输时可以将文件缓冲区的数据直接发送到目的通道（Channel）  
	- ByteBuf 可以通过 wrap 操作把字节数组、ByteBuf、ByteBuffer 包装成一个 ByteBuf 对象, 进而避免了拷贝操作  
	- ByteBuf 支持 slice 操作, 因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝
	- Netty 提供了 CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免了各个 ByteBuf 之间的拷贝
	>其中第 1 条属于操作系统层面的零拷贝操作，后面 3 条只能算用户层面的数据操作优化。
	
* RocketMQ：mmap + write
* Kafka：sendfile
### 零拷贝在消息队列中的应用

RocketMQ 选择了 **mmap + write** 这种零拷贝方式，适用于业务级消息这种小块文件的数据持久化和传输；而 Kafka 采用的是 **sendfile** 这种零拷贝方式，适用于系统日志消息这种高吞吐量的大块文件的数据持久化和传输。但是值得注意的一点是，**Kafka 的索引文件使用的是 mmap + write 方式，数据文件使用的是 sendfile 方式**。

| 消息队列 | 零拷贝方式 | 优点 | 缺点 |
| ------  | ------ | ------ | ------ |
| RocketMQ | mmap + write | 适用于小块文件传输，频繁调用时，效率很高 | 不能很好的利用DMA方式，会比sendfile多消耗CPU，内存安全性控制复杂，需要避免JVM Crash问题 |
| Kafka | sendfile | 可以利用DMA方式，消耗CPU较少，大块文件传输效率高，无内存安全性问题 | 小块文件传输效率低于mmap方式，只能是BIO传输方式，不能使用NIO方式 |