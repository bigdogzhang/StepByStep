# IPC共通

不论是什么OS，不论是什么类型的通信方式，也不论是进程通信，还是线程通信，本质上都是开辟一块内存，由通信双方共同访问，通过共享实现通信。通信方式的差异，本质上有两点：

- 通信双方如何访问此内存块，即：如何为进程设定接口。PIPE使用文件描述符，在进程间继承。FIFO使用文件名，在进程间共享打开。消息队列提供内核级别的IPC描述符。网络Socket使用套接字描述符。


- 通信双方的应用需求。根据不同的应用需求，创建或选择不同的通信方式，可以提高程序开发的效率。比如：T-Kernel中的Message Buffer与Mail Box。消息队列传送有边界的消息，管道&FIFO传送无边界的字符，共享内存满足了一个写者，多个读者的需求。网络Socket可以实现计算机之间的进程通信。

除了共享内存外，管道，FIFO，消息队列，网络Socket，都有写入端和读出端。设定了非Block模式时，当内部Buffer空时，读出端被阻塞；当内部Buffer满时，写入端被阻塞。除了消息队列外，管道，FIFO，网络Socket的写入端和读出端都可以分别关闭（Close），任何一端被关闭后，另一端都会得到通知。读出端被关闭后，写入端会返回错误；写入端被关闭后，读出端会返回0。特殊说明的是，FIFO的写入端和读出端关闭，指的是写入进程或者读出进程关闭FIFO文件。另外，打开FIFO文件时，如果另一端不存在，则Open处理会一直阻塞到另一端出现。

## 管道&FIFO

管道只能在具有公共祖先的两个进程之间使用，这是由管道的使用方式决定的。管道被创建后，对外提供的接口只有两个整型的文件描述符（fd[2]），进程使用文件描述符，调用read和write方法访问管道。然而，文件描述符是进程相关的变量，每个进程维护一张文件描述符表（FD Table），通过文件描述符，找到其对应的内核文件表，进而访问其对应的文件。文件描述符表只能在父子进程之间继承（拷贝），其他进程之间，仅仅获得文件描述符，进程内部的FD表中如果没有相关记录，进程也不能访问该文件（管道）。所以，严格来说，具有公共祖先的两个进程之间也未必一定可以使用管道，管道涉及的文件描述符只有在两个进程之间有继承拷贝关系，两个进程才可以使用管道通信。最后一个访问管道的进程结束后，管道文件描述符都将被close，对应的内核文件表的表项也被删除，管道彻底被清除。

FIFO之所以被称为命名管道，是因为FIFO本质上是文件，只要知道了文件名字，就可以使用open函数返回对应的文件描述符，进而使用read和write方法访问FIFO。注意：FIFO本质上是文件，也可以使用标准IO函数访问。因为文件名字是全局的，所以FIFO不存在管道那样的限制，可以在任意进程之间使用。管道文件可以使用mkfifo函数生成，也可以在shell中调用mkfifo命令生成。

除了上面的区别，PIPE与FIFO可以认为是相同的东西。

Read端示例代码：

```c
  1 #include <stdio.h>
  2 #include <fcntl.h>
  3 #include <unistd.h>
  4 
  5 #define MAXLINE 250
  6 
  7 int main()
  8 {
  9         int fd = open("test_fifo", O_RDONLY);
 10         if (fd > 0)
 11         {
 12                 char buff[MAXLINE] = {0};
 13                 read(fd, buff, sizeof(buff));
 14                 printf("read from server: %s\n", buff);
 15         }
 16 
 17         return 0;
 18 }
```

Write端示例代码：

```c
  1 #include <stdio.h>
  2 
  3 int main()
  4 {
  5         FILE *fp = fopen("test_fifo", "w");
  6         if (fp != NULL)
  7         {
  8                 fprintf(fp, "%s\n", "Hello server again.");
  9                 fclose(fp);
 10         }
 11 
 12         return 0;
 13 }
```

### PIPE_BUF & Pipe Capacity

管道有两个重要概念，PIPE_BUF和管道容量。PIPE_BUF是原子写操作的最大容量（通常4K），超出该限制的写操作，不保证原子性，即：多进程同时写一个管道时，可能出现数据交叉。程序中，要么保证每次写的数据不超过PIPE_BUF，要么进行进程互斥控制。详细可参考下文：

<http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html>

> Write requests to a pipe or FIFO shall be handled in the same way as a regular file with the following exceptions:
>
> - There is no file offset associated with a pipe, hence each write request shall append to the end of the pipe.
> - Write requests of {PIPE_BUF} bytes or less shall not be interleaved with data from other processes doing writes on the same pipe. Writes of greater than {PIPE_BUF} bytes may have data interleaved, on arbitrary boundaries, with writes by other processes, whether or not the O_NONBLOCK flag of the file status flags is set.
> - If the O_NONBLOCK flag is clear, a write request may cause the thread to block, but on normal completion it shall return *nbyte*.
> - If the O_NONBLOCK flag is set, *write*() requests shall be handled differently, in the following ways:
>   - The *write*() function shall not block the thread.
>   - A write request for {PIPE_BUF} or fewer bytes shall have the following effect: if there is sufficient space available in the pipe, *write*() shall transfer all the data and return the number of bytes requested. Otherwise, *write*() shall transfer no data and return -1 with *errno* set to [EAGAIN].
>   - A write request for more than {PIPE_BUF} bytes shall cause one of the following:
>     - When at least one byte can be written, transfer what it can and return the number of bytes written. When all data previously written to the pipe is read, it shall transfer at least {PIPE_BUF} bytes.
>     - When no data can be written, transfer no data, and return -1 with *errno* set to [EAGAIN].

管道容量，指的是管道可接受的最大容量（通常64K），本质上就是为管道开辟的内存容量，超出该容量的数据将无法写入，使用fcntl函数可以改变管道容量（F_SETPIPE_SZ）。

## 共享内存

在共享内存中建立链表，不能直接保存实际的物理地址，因为服务器进程和客户进程可能会将共享内存映射到不同的地址（进程空间地址）。当在共享内存中建立链表时，链表指针的值会设置为共享内存内对象的偏移量。偏移量为所指对象的实际地址减去共享内存的起始地址。

## 网络Socket

### 面向连接与无连接的区别

无连接的通信双方不需要事先建立逻辑连接，每个数据报文都包含独立的通信地址，各个数据报文之间彼此没有关系，数据报文也没有固定的传送顺序。面向连接的通信双方需要事先建立逻辑连接，通信数据中不包含对方的通信地址，连接本身暗示了通信的源和目的地。

SOCK_DGRAM：无序的，固定长度的，无连接的，不可靠的数据报文传递。

SOCK_STREAM：有序的，可靠的，双向的，面向连接的字节流。

注意几个概念的区别：有序与无序；可靠与不可靠；面向连接与无连接；数据报文与字节流。

面向连接的服务，为每个传输的数据都做了编号，因此可以保证发送顺序与接受顺序是相同的，同时，某个数据丢失后，可以通过重传的方式实现可靠服务。无连接服务，因为没有逻辑连接，无法为每个传输数据做编号，所以无法提供有序的，可靠的服务，除非应用层自己去实现。

### 函数shutdown与close的区别

socket通信是双向的，可以使用shutdown关闭双向传输中的一个方向，同时让另一个方向继续保持传输。close是彻底释放该套接字，释放网络端点。注意：套接字本质上是文件描述符，所以可以采用dup进行复制，只有最后一个文件描述符都被释放后，该套接字对应的网络端点才最终被释放。而shutdown本质上是操作套接字对应的网络端点，与套接字被引用（复制）的数目无关。不论是shutdown还是close，一旦调用了，就不能reopen了，这是套接字的限制，也是通信协议的限制。

套接字虽然本质上是文件描述符，操作文件描述符的函数也可以被应用在套接字上，但是动作并不完全相同。除非希望复用原本为处理本地文件而设计的函数，或者，希望通过文件描述符屏蔽网络socket的存在，从而使程序有更大的通用性，还是尽可能使用套接字的接口函数操作套接字。

### 绑定Bind

socket被创建后，并不包含地址信息，需要绑定到一个地址。一个地址标识一个特定通信域的套接字端点，地址格式与这个特定的通信域相关。为使不同格式地址能够传入到套接字函数，地址会被强制转换成一个通用的地址结构sockaddr。后面说的地址，均指的是socket地址sockaddr，其包含了网络类型，IP地址，端口号等信息。客户端在调用connect之前，可以不执行绑定动作，此时系统会选择一个默认的地址（含端口）与socket进行绑定。服务端必须调用bind函数，使服务器套接字与一个地址关联。当然，客户端也可以执行bind，使socket与指定的地址和端口进行绑定。通过getsockname函数和getpeername函数，可以通过socket号获得绑定的地址信息，和通信对方的地址信息。

### 连接connect

connect可能会返回错误，这些错误可能是由一些瞬时条件引起的。如果一个服务器运行在一个负载很重的系统上，就很有可能发生这些错误。因此，connect返回错误后，不要立刻结束程序，必须有retry处理。但是，需要注意的是，某些系统的connect失败后，套接字的状态会变成未定义的，因此，可移植的应用程序需要关闭出错的套接字，再打开一个新的套接字，继续connect。

通常，只有面向连接的socket才需要connect，但是无连接的报文服务也是可以调用connect的。如果用SOCK_DGRAM套接字调用connect，传送的报文的目标地址会设置成connect调用中所指定的地址，这样每次传送报文时就不需要再提供地址。另外，仅能接收来自指定地址的报文。

通过accept建立的socket，不是传递给accept的socket。新建立的socket连接了服务端和客户端，由服务端IP，服务端端口，客户端IP，客户端端口，协议号组成，任何一个不一致，都表示不同的socket。传递给accept的socket，没有关联到客户端的连接，继续保持可用状态并接收其他连接请求。两者的作用不同。

这也就可以理解，同一个端口可以接受多个连接请求，因为socket是由五个因素区分的（服务端IP，服务端端口，客户端IP，客户端端口，协议号组成）。但是注意，多个连接请求一定不会是同一个IP地址的同一端口号，因为无法区分socket了。在服务器端，可以通过简单的计数，控制一个端口号可以接受多少个连接，也可以通过accept得到客户端的地址信息，借此判断是否接受这个请求，比如：只接受某个端口号的请求。

另外，关于listen函数的backlog（积压），可以参考下面的说明进行设定。

<http://tangentsoft.net/wskfaq/advanced.html#backlog>

> ##### 4.14 - What is the connection backlog?
>
> When a connection request comes into a [network stack](http://tangentsoft.net/wskfaq/glossary.html#stack), it first checks to see if any program is listening on the requested port. If so, the stack replies to the [remote peer](http://tangentsoft.net/wskfaq/glossary.html#peer), completing the connection. The stack stores the connection information in a queue called the connection backlog. (When there are connections in the backlog, the `accept()` call simply causes the stack to remove the oldest connection from the connection backlog and return a socket for it.)
>
> One of the parameters to the `listen()` call sets the size of the connection backlog for a particular socket. When the backlog fills up, the stack begins rejecting connection attempts.
>
> Rejecting connections is a good thing if your program is written to accept new connections as fast as it reasonably can. If the backlog fills up despite your program’s best efforts, it means your server has hit its load limit. If the stack were to accept more connections, your program wouldn’t be able to handle them as well as it should, so the client will think your server is hanging. At least if the connection is rejected, the client will know the server is too busy and will try again later.
>
> The proper value for `listen()`’s `backlog` parameter depends on how many connections you expect to see in the time between `accept()` calls. Let’s say you expect an average of 1000 connections per second, with a burst value of 3000 connections per second. (I picked these values because they’re easy to manipulate, not because they’re representative of the real world!) To handle the burst load with a short connection backlog, your server’s time between `accept()` calls must be under 0.3 milliseconds. Let’s say you’ve measured your time-to-accept under load, and it’s 0.8 milliseconds: fast enough to handle the normal load, but too slow to handle your burst value. In this case, you could make `backlog` relatively large to let the stack queue up connections under burst conditions. Assuming that these bursts are short, your program will quickly catch up and clear out the connection backlog.
>
> The traditional value for `listen()`’s `backlog` parameter is 5. This is actually the limit on the home and workstation class versions of Windows. On Windows Server, the maximum connection backlog size is 200, unless the dynamic backlog feature is enabled. (More info on dynamic backlogs below.) Because the stack will use its maximum backlog value if you pass in a larger value, you can pass a special constant, `SOMAXCONN`, to `listen()` which tells it to use whatever the platform maximum is, since the constant’s value is 0x7FFFFFFF. There is no standard way to find out what backlog value the stack chose to use if it overrides your requested value.
>
> If your program is quick about calling `accept()`, low backlog limits are not normally a problem. However, it does mean that concerted attempts to make lots of connections in a short period of time can fill the backlog queue. This makes non-Server flavors of Windows a bad choice for a [high-load server](http://tangentsoft.net/wskfaq/intermediate.html#high-load): either a legitimate load or a `SYN` flood attack can overload a server on such a platform. (See below for more on `SYN` attacks.)
>
> Beware that large backlogs make `SYN` flood attacks much more, shall we say, effective. When Winsock creates the backlog queue, it starts small and grows as required. Since the backlog queue is in non-pageable system memory, a `SYN` flood can cause the queue to eat a lot of this precious memory resource.
>
> After the first `SYN` flood attacks in 1996, Microsoft added a feature to Windows NT 4.0 SP3 called the “dynamic backlog.” This feature is normally off for backwards compatibility, but when you turn it on, the stack can increase or decrease the size of the connection backlog in response to network conditions. (It can even increase the backlog beyond the normal maximum of 200, in order to soak up malicious `SYN`s.) The Microsoft Knowledge Base [article](http://support.microsoft.com/kb/142641) that describes the feature also has some good practical discussion about connection backlogs.
>
> You will note that `SYN` attacks are dangerous for systems with both very short and very long backlog queues. The point is that a middle ground is the best course if you expect your server to withstand `SYN` attacks. Either use Microsoft’s dynamic backlog feature, or pick a value somewhere in the 20-200 range and tune it as required.
>
> A program can rely too much on the backlog feature. Consider a single-threaded blocking server: the design means it can only handle one connection at a time. However, it can set up a large backlog, making the stack accept and hold connections until the program gets around to handling the next one. (See [this example](http://tangentsoft.net/wskfaq/examples/basics/basic-server.html)to see the technique at work.) You should not take advantage of the feature this way unless your connection rate is very low and the connection times are very short. (Pedagogues excepted. Everyone else designing such a program should probably use 1, 0, or even close the listening socket while there is a client connected.)



