# IPC共通

不论是什么OS，不论是什么类型的通信方式，也不论是进程通信，还是线程通信，本质上都是开辟一块内存，由通信双方共同访问，通过共享实现通信。通信方式的差异，本质上有两点：

- 通信双方如何访问此内存块，即：如何为进程设定接口。PIPE使用文件描述符，在进程间继承。FIFO使用文件名，在进程间共享打开。消息队列提供内核级别的IPC描述符。
- 通信双方的应用需求。根据不同的应用需求，创建或选择不同的通信方式，可以提高程序开发的效率。比如：T-Kernel中的Message Buffer与Mail Box。

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