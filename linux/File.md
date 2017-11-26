# 文件IO相关

在linux下，所有对象本质上都是文件。

有两个文件相关的概念：文件描述符和文件指针（FILE *）

## 文件描述符

文件描述符是整型。linux内核维护一个Kernel File Table（简称文件表，但具体可能由多个结构体实现），任何进程打开的任何文件，都由这个表维护，每个打开的文件对应一个表项（注意：同一个文件被不同的进程打开，Kernel File Table会维护两个不同的表项，因为打开文件的模式，以及访问文件的位置信息等是不同的，但是最终肯定指向了同一个物理文件）。每个表项可能纪录了下面的信息：

- Current File Position
- inode info
- vnode info
- file metadata
- etc

同时，每个进程也会维护一个File Descriptor(FD)表格，其记录了进程中所有使用的文件描述符。文件描述符的编号是独立于进程的，由内核维护。内核永远选择当前可用数值中最小的文件描述符返回给进程。每个文件描述符都指向了Kernel File Table中的某个表项，多个文件描述符可能指向同一个文件表项，此时，多个文件描述符共享该文件的访问状态。FD表格可以从父进程继承，父子进程将共享拷贝过来的文件描述符，共享文件的访问状态。

dup(fd)可以在当前进程中复制文件描述符，即：在进程的FD表格中增加一个新的文件描述符（由内核返回，数值是当前可用数值中最小的）。但是，复制的FD肯定指向了同一个文件表，复制源FD与复制目标FD共享文件的访问状态。

dup2(fd, fd2)本质上与dup(fd)相同，只不过可以指定复制目标的FD数值。例如：dup2(fd, STDIN_FILENO)可以让标准输入用的文件描述符指向fd对应的文件表，从而实现输入重定向。

open函数返回的FD，由close函数关闭。注意：其仅仅是关闭了该FD而已，并不一定会同时删除相应的文件表，如果还有其他FD也指向了同一个文件表，那么该文件表将仍然可用。

## 文件指针

文件指针是FILE结构体指针。FILE结构体面向应用程序，由ISO的stdio定义，OS内核并不参与，不同的OS，FILE结构体的定义也不同。下面是Linux的FILE结构体定义：</usr/include/stdio.h>

```c
 * code as compact as possible.
 *
 * _ub, _up, and _ur are used when ungetc() pushes back more characters
 * than fit in the current _bf, or when ungetc() pushes back a character
 * that does not match the previous one in _bf.  When this happens,
 * _ub._base becomes non-nil (i.e., a stream has ungetc() data iff
 * _ub._base!=NULL) and _up and _ur save the current values of _p and _r.
 *
 * NB: see WARNING above before changing the layout of this structure!
 */
typedef struct __sFILE {
        unsigned char *_p;      /* current position in (some) buffer */
        int     _r;             /* read space left for getc() */
        int     _w;             /* write space left for putc() */
        short   _flags;         /* flags, below; this FILE is free if 0 */
        short   _file;          /* fileno, if Unix descriptor, else -1 */
        struct  __sbuf _bf;     /* the buffer (at least 1 byte, if !NULL) */
        int     _lbfsize;       /* 0 or -_bf._size, for inline putc */

        /* operations */
        void    *_cookie;       /* cookie passed to io functions */
        int     (* _Nullable _close)(void *);
        int     (* _Nullable _read) (void *, char *, int);
        fpos_t  (* _Nullable _seek) (void *, fpos_t, int);
        int     (* _Nullable _write)(void *, const char *, int);

        /* separate buffer for long sequences of ungetc() */
        struct  __sbuf _ub;     /* ungetc buffer */
        struct __sFILEX *_extra; /* additions to FILE to not break ABI */
        int     _ur;            /* saved _r when _r is counting ungetc data */

        /* tricks to meet minimum requirements even when malloc() fails */
        unsigned char _ubuf[3]; /* guarantee an ungetc() buffer */
        unsigned char _nbuf[1]; /* guarantee a getc() buffer */

        /* separate buffer for fgetln() when line crosses buffer boundary */
        struct  __sbuf _lb;     /* buffer for fgetln() */

        /* Unix stdio files get aligned to block boundaries on fseek() */
        int     _blksize;       /* stat.st_blksize (may be != _bf._size) */
        fpos_t  _offset;        /* current lseek offset (see WARNING) */
} FILE;
```

FILE结构体包含了文件描述符`short _file`，增加了文件访问缓冲等处理。基于文件描述符的文件访问（如：read, write）是不带缓冲的，而基于FILE结构体的访问（如：fread, fwrite）是可以带缓冲的。

```c
1. File *fp;
2. fp = fopen ("/etc/myfile.txt", "w");
3. fclose(fp);
```

语句1: 在栈上创建4字节的fp指针。

语句2: 在堆上申请了sizeof(FILE)大小的空间，地址赋给fp。同时，Kernel File Table中增加一个表项，记录了本次打开的文件。在进程FD表中，也同时增加一个FD表项，文件表指针指向Kernel File Table中的新增项。对文件的任何读写操作（如：read, write, fread, fwrite等），文件访问状态由内核维护。

语句3: 关闭堆上申请的内存。删除Kernel File Table和进程FD表中的记录。

上面的动作由C运行库完成，内核并不参与。

## 两者的主要区别

文件描述符是底层OS相关的概念，对文件的访问最终都通过文件描述符完成。但是，文件描述符的可移植性差。

文件指针是ISO定义的面向应用层的概念，可移植性更好一些。

两者没有性能上的差别，根据具体需要选择使用。

fileno函数：从文件指针得到文件描述符。

fdopen函数：从文件描述符得到文件指针。