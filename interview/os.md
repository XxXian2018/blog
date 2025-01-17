# interview thirteen：os

- 同步/异步、阻塞/非阻塞
- I/O模型：同步阻塞模型、同步非阻塞模型、异步模型
- select()/poll()和epoll()的区别

# **1. malloc()**

> 不同进程中malloc()函数返回的值会是相同的吗？（会，因为有虚拟内存）

# **2. mmap()**

![mmap](https://asea-cch.life/upload/2022/02/mmap-5f8b3ca0955844efb19e27e7fb5733b2.png)

![虚拟内存与内存映射](https://asea-cch.life/upload/2022/02/%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E4%B8%8E%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84-4ddd478a39d5448894d7ec40e17b88e8.png)

- mmap()：内核函数调用，用于向**进程空间结构task_struct.mm_sturct**的vma中分配一段**虚拟空间**

    虚拟空间通过**MMU单位**和**内核页表**建立真实物理内存映射，**物理内存再与文件描述符建立映射**，这段物理空间也可以认为是kernel page cache的一部分

- page cache：又称kernel buffer，用于弥补磁盘与内存之间的读写速度差异提升性能，注意要跟磁盘自带的高速cache作区分

结论：mmap()调用后，内核分配某一块真实物理内存page cache和进程虚拟空间映射，并将该page cache与文件进行映射，当文件数据到达page cache后，进程可直接感知数据并通过指针操作，无须进行内存拷贝

> 换言之，避免从page cache拷贝到用户进程堆内存的开销

流程：

1. 进程陷入内核态，调用mmap()函数分配虚拟内存

2. 内核为进程分配虚拟内存，并在页表中添加page cache与进程虚拟内存的映射关系后，建立文件描述符与虚拟内存对应物理内存的映射

3. 当进程读取文件数据时触发缺页异常，磁盘读取数据直至完毕，向DMA发出中断信号

4. DMA将数据从磁盘高速cache拷贝到page cache中

5. 进程虚拟内存可直接感知到page cache数据，通过指针操作数据

# **3. 为什么用协程不用线程（线程一定比协程好吗？）**

# **4. 进程调度有哪些算法**

> Linux调度用了什么算法？

# **5. Linux里进程通信有几种方式**

# **6. 进程同步有几种方式**

# **7. 管程**

# **8. 操作系统的栈和队列有哪些应用场景**

# **同步/异步、阻塞和非阻塞**

> 如何理解同步和阻塞

以`进程`视角分为两个步骤：

1. 发起I/O请求

2. 实际的I/O读写操作

以`操作系统`视角分为两个步骤：

1. kernel数据准备，将数据从**磁盘缓冲区搬运都内核缓冲区**（DMA）

2. 将数据从内核空间拷贝到进程空间中（CPU）

- 同步、异步：指是否由**进程自身占用CPU**，将数据从内核拷贝到进程空间中，是的话为同步，否则为异步

    - 异步：由**内核占用CPU**将数据拷贝到进程中，完成后通知进程，进程在此期间可以处理自己的任务

- 阻塞、非阻塞：指进程在**发起I/O请求**后是否处于阻塞状态

    - 阻塞I/O模型：同步阻塞I/O

    - 非阻塞I/O模型：同步非阻塞I/O（包括I/O多路复用）、异步I/O

    BIO会阻塞，NIO不会阻塞，`对于NIO而言通过channel进行IO操作请求后，其实就返回了`

# **I/O模型**

1. 同步阻塞I/O

    进程视角：进程在发起I/O请求后**阻塞至I/O读写操作完毕**
    
    系统视角：等待kernel数据完毕后，**进程自身占用cpu拷贝**

    数据由DMA从磁盘/网卡搬运到内核空间中，再由中断信号唤醒进程，进程占用CPU将数据从内核空间拷贝到用户空间

2. 同步非阻塞I/O：可直接代指I/O多路复用

    进程视角：进程在发起I/O请求后**可以立即返回**
    
    系统视角：当kernel数据准备完毕后，需**进程自身占用cpu拷贝**

    `I/O多路复用`：进程发起I/O请求后可以立即返回，由一个I/O线程（selector）检测多个注册在其上句柄的就绪状态：

    - select、poll：将进程加入到socket的等待队列中，由socket接收完毕数据后唤醒

    - epoll：引入eventpoll epfd作为进程与socket的中介，socket完毕后会调用中断程序使得rdlist持有socket的引用，实现句柄就绪后的通知

3. 异步I/O

    进程视角：进程在发起I/O请求后不会进入阻塞状态
    
    系统视角：kernel数据准备完毕后，由**内核占用cpu拷贝**到进程空间，完成后通知进程

    - CPU密集型系统：表现不佳，因为内核占用cpu势必导致进程的cpu可分配资源降低，会增加该类型系统计算延时

    - I/O密集型系统：表现佳（如node.js、aio）

# **select()与epoll()的区别**

都用于**监视多个socket句柄**的状态，epoll相比select更加高效

- select()：

    1. 从进程空间向内核空间传入socket列表FD_SET，遍历FD_SET后将进程加入到各fd的等待队列中，selector进入阻塞状态

        第一次遍历FD_SET，用于将进程加入到对于socket的等待队列中，涉及FD_SET到内核的内存拷贝

    2. 某socket接收到数据，发出cpu中断信号，cpu执行中断程序进行数据搬运，并唤醒进程

        - 数据搬运：数据从网卡缓冲区搬运到kernel socket buffer

            > DMA：进行 I/O 设备和内存的数据传输的时候，数据搬运的工作交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情

            优化后步骤：某socket接收到数据后，**向DMA发送中断信号**，由DMA负责数据搬运而不占用CPU。DMA完成搬运后，再向CPU发起中断信号，`CPU执行中断程序后，再唤醒进程`

        - 唤醒进程：将进程从所有socket fd的等待队列中移除

        第二次遍历FD_SET，从而得知哪些socket等待队列上有该进程，所以该次遍历**同样需要**将FD_SET传递到内核

    3. 进程只知道至少有一个socket接收了数据，需要遍历FD_SET获取就绪的fd，并根据可读/可写事件调用read()/write()

        第三次遍历FD_SET，这次遍历在进程用户空间进行，主要是获知已就绪的fd

        NIO：通过mmap() 直接打通了kernel socket buffer到用户空间，无需内存拷贝，具体实现是使用DirectByteBuffer（extends `MappedByteBuffer`）

- epoll()：

    1. 通过epoll()的函数对fd进行注册

    - epoll_create()：创建epfd（eventpoll）
    
        - epfd：本身就是一个红黑树，用于存储注册在其上的句柄，从而**避免每次调用都需要传入fd集合产生拷贝**
        
        - epfd.rdlist：epfd包含一个双向链表实现的就绪队列，当有socket的数据就绪后，将由DMA、CPU配合执行中断程序将就绪fd加入到队列中，从而避免每次进程还需重新遍历fd集合获取就绪的fd
    
    - epoll_ctl()：添加需要监视的socket fd到epfd中，**内核会将eventpoll加入到这些socket的等待队列**

    - epoll_wait()：进程进入epfd的等待队列中，进入阻塞状态

        监视的socket fd基本不会变动，后续无需每次都进行注册，进一步减少了第一次遍历

    2. socket接收到数据后，向DMA发起中断信号。DMA读取数据完毕后，向CPU发起中断信号执行中断程序，cpu将执行**设备驱动回调**，将就绪的文件描述符加入到epfd的rblist就绪队列中

        epfd作为进程与socket的中介者，socket的等待队列将一直持有eventpoll引用，无需移出队列减少了select第二次遍历

    3. 驱动将就绪socket fd加入rdlist，并唤醒在epfd等待队列上的进程

        进程可直接在rdlist中获取到就绪fd，减少了select的第三次遍历

总结：

1. 句柄数增多导致的性能问题

    - select()/poll()：需要对FD_SET进行三次线性遍历，在句柄数越多时表现越差，select最多监视1024个，poll使用链表结构可以监视更多句柄，但依旧没有改善效率低的问题

    - epoll()：没有影响，因为是通过注册设备回调实现就绪列表rblist，只有活跃的socket才会主动调用，进程可以直接从就绪队列获取到就绪

2. 消息传递方式

    - select()：每次调用select()都会将FD_SET从用户空间中拷贝到内核空间

    - epoll()：fd存放在内核eventpoll中，在**用户空间和内核空间的copy仅需一次**

    > epoll没有使用mmap！

3. 监视方式：都在阻塞状态中由中断信号唤醒，select()由socket直接唤醒，epoll()由socket间接通过eventpoll唤醒

# **ET（边缘触发）和LT（水平触发）**



# 参考
- [腾讯WXG | 一二+面委+HR|已拿offer](https://leetcode-cn.com/circle/discuss/ON7r4A/)

# 重点参考
- [Linux虚拟内存和内存映射（mmap）](https://www.cnblogs.com/linguoguo/p/15807313.html)
- [共享内存和文件内存映射的区别](https://zhuanlan.zhihu.com/p/149277008)
- [关于共享内存shm和内存映射mmap的区别是什么？ - 妄想者的回答 - 知乎](https://www.zhihu.com/question/401612303/answer/1428608073)
- [Linux中的mmap映射 [一] - 兰新宇的文章 - 知乎](https://zhuanlan.zhihu.com/p/67894878)