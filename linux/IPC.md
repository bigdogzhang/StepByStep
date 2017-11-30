# 管道&FIFO

管道只能在具有公共祖先的两个进程之间使用，这是由管道的使用方式决定的。管道被创建后，对外提供的接口只有两个整型的文件描述符（fd[2]），进程使用文件描述符，调用read和write方法访问管道。然而，文件描述符是进程相关的变量，每个进程维护一张文件描述符表（FD Table），通过文件描述符，找到其对应的内核文件表，进而访问其对应的文件。文件描述符表只能在父子进程之间继承（拷贝），其他进程之间，仅仅获得文件描述符，进程内部的FD表中如果没有相关记录，进程也不能访问该文件（管道）。所以，严格来说，具有公共祖先的两个进程之间也未必一定可以使用管道，管道涉及的文件描述符只有在两个进程之间有继承拷贝关系，两个进程才可以使用管道通信。最后一个访问管道的进程结束后，管道文件描述符都将被close，对应的内核文件表的表项也被删除，管道彻底被清除。

FIFO之所以被称为命名管道，是因为FIFO本质上是文件，只要知道了文件名字，就可以使用open函数返回对应的文件描述符，进而使用read和write方法访问FIFO。注意：FIFO本质上是文件，也可以使用标准IO函数访问。因为文件名字是全局的，所以FIFO不存在管道那样的限制，可以在任意进程之间使用。管道文件可以使用mkfifo函数生成，也可以在shell中调用mkfifo命令生成。

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

