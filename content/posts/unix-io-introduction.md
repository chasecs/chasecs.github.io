+++
title = "Unix I/O 主要模型介绍"
date = 2020-09-24
images = []
tags = []
categories = []
+++
>- 5 种 I/O 模型及一些示例
>- 目前主流的 I/O 方案 epoll / kqueue

<!--more-->


在 Unix 系统里，磁盘的文件读写、网络通讯等功能的核心部分都是 I/O， open， read ， write ，connect 等系统调用都属于 I/O 操作。因为 I/O 操作都比较耗时，因此有了阻塞、非阻塞、同步、异步等不同 I/O 模型。

I/O 操作引发的阻塞是指进程中断执行，等待其它的工作（通常是比较耗时的）准备就绪再恢复。阻塞可能出现在两个阶段：

- 阶段一：等待数据就绪（例如从硬盘读取数据完成）
- 阶段二：把数据从内核复制到用户空间

按照《Unix Network Programming》（下称 UNP）的分类，Unix 有 5 种 I/O 模型：

- 阻塞式 I/O （ Blocking I/O）
- 非阻塞式 I/O （NonBlocking I/O）
- I/O 复用 （I/O multiplexing）
- 信号驱动 I/O ( signal-driven I/O )
- 异步 I/O （asynchronous I/O )

他们在上述两个阶段的阻塞情况和处理方式有所差别。

![graphics/06fig06.gif](http://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/files/06fig06.gif)

UNP对I/O模型的分类比较严格，认为只有前4个都属于同步的（synchronous），只有第5个属于异步（asynchronous）。

不过，目前网上也有很多说法认为把 I/O 复用，epoll，kqueue 归为异步 I/O，如下表：

|   | Blocking   | 非阻塞    |
| -----| -------- | ---------- |
| Synchronous  | read/write                          | read/write(O_NONBLOCK) |
| Asynchronous | I/O multiplexing(select/poll/epoll) | AIO                    |

虽然在分类上有不通的说法，但弄清楚 I/O 最关键的阻塞、非阻塞等概念，对于理解各种不同的 I/O 方案就没有太大问题。


##  阻塞式 I/O （ Blocking I/O）

阻塞式是最原始的 I/O 模型，默认情况下，包括 read、 write、connect、accept 等的 I/O 操作默认都是阻塞式的。

阻塞式 I/O，当系统调用（如下面代码中的 `write` )执行后，进程会停止继续执行，直到该系统调用完成工作（`write`写入数据），进程才继续执行之后的代码。

```c
#include "apue.h"
#include <errno.h>
#include <time.h>

char buf[500000];

int main(void)
{
    int ntowrite, nwrite;
    char *ptr;

    ntowrite = read(STDIN_FILENO, buf, sizeof(buf));
    fprintf(stderr, "read %d bytes\n", ntowrite);

    ptr = buf;
    while (ntowrite > 0)
    {
        errno = 0;
        clock_t begin = clock();
	      //write is blocking
        nwrite = write(STDOUT_FILENO, ptr, ntowrite);
        clock_t end = clock();
				//counting time consumed by write() 
        double time_spent = (double)(end - begin) / CLOCKS_PER_SEC;
        printf("\nnwrite = %d, errno = %d, time = %f\n", nwrite, errno, time_spent);

        if (nwrite > 0)
        {
            ptr += nwrite;
            ntowrite -= nwrite;
        }
    }

    exit(0);
}
```

编译上述代码，然后找一个测试文件运行

```bash
$ gcc -o blk blocking.c
$ ./blk < test_file
```

可以看到输出以下结果（nwrite 的数值因测试文件的不同有差异）：

```bash
nwrite = 30015, errno = 0, time = 0.000371
```

可以看到 write 花费的时间是  0.000371 秒，在这段时间内，进程是阻塞的。

**阻塞式 I/O 很难应对高并发的应用场景**，因为每次阻塞，都有一个进程被搁置起来，直到完成工作，才会去处理下一个请求。

  

## 非阻塞式 I/O （NonBlocking I/O）

在非阻塞式模型下，I/O 操作启动后，如果工作还未完成，内核会马上返回一个错误消息（EWOULDBLOCK或者EAGAIN 等) ，而不是像阻塞式一样等到完成工作再交出进程。

非阻塞式 I/O 可以通过 `fcntl` 函数修改“文件描述符”实现， 例如改造上文例子的 `STDOUT_FILEN`

```c
flags = fcntl(STDOUT_FILEN, F_GETFL);
flags &= ~O_NONBLOCK;
fcntl(STDOUT_FILEN, F_SETFL, flags);
```

改成非阻塞之后的主要代码如下

```c
int main(void)
{

    ...

    set_fl(STDOUT_FILENO, O_NONBLOCK); /* set nonblocking */

    ptr = buf;
    while (ntowrite > 0)
    {
        errno = 0;
        clock_t begin = clock();
        //write is non-blocking now
        nwrite = write(STDOUT_FILENO, ptr, ntowrite);
        clock_t end = clock();
        double time_spent = (double)(end - begin) / CLOCKS_PER_SEC;
        fprintf(stderr, "nwrite = %d, errno = %d, time = %f\n", nwrite, errno, time_spent);

				...
    }

    clr_fl(STDOUT_FILENO, O_NONBLOCK); /* clear nonblocking */

    exit(0);
}

void set_fl(int fd, int flags) /* flags are file status flags to turn on */
{
    int val;

    if ((val = fcntl(fd, F_GETFL, 0)) < 0)
        printf("fcntl F_GETFL error");

    val |= flags; /* turn on flags */

    if (fcntl(fd, F_SETFL, val) < 0)
        printf("fcntl F_SETFL error");
}

void clr_fl(int fd, int flags) /* flags are file status flags to turn off */
{
    int val;

    if ((val = fcntl(fd, F_GETFL, 0)) < 0)
        printf("fcntl F_GETFL error");

    val &= ~flags; /* turn flags off */

    if (fcntl(fd, F_SETFL, val) < 0)
        printf("fcntl F_SETFL error");
}
```

编译上述代码，然后找一个测试文件运行

```bash
$ gcc -o nblk nonblocking.c
$ ./nblk < test_file 2>log
```

打开 log 日志可以看到类似下面的结果

```bash
read 30015 bytes
nwrite = 989, errno = 0, time = 0.000019
nwrite = -1, errno = 35, time = 0.000002
nwrite = -1, errno = 35, time = 0.000002
....
nwrite = 995, errno = 0, time = 0.000006
nwrite = -1, errno = 35, time = 0.000002
....
nwrite = -1, errno = 35, time = 0.000001
nwrite = 993, errno = 0, time = 0.000006
..
```

在非阻塞模式下，每次write 的响应时间快了很多，没有数据的时候会返回 errno =35 (对应 EAGAIN），然后去轮询 write 的结果。**这种非阻塞模式，对 CPU 的消耗比较大，因此并没有被广泛应用。**



##  I/O 复用 （I/O multiplexing）

multiplexing 使用 select、 poll 等函数实现，**使阻塞发生在 select 等函数里，而不是阻塞进程。**

select 可以**同时监听多个文件描述符**，它执行后会处于等待状态，当有一个或者多个文件描述符变成“就绪”状态，就会继续向下执行，如此循环。

以下是使用 select 处理 socket I/O 的代码片段：

```c
		int fds[5];
    fd_set rset;
    for (i = 0; i < 5; i++)
    {
        memset(&client, 0, sizeof(client));
        addrlen = sizeof(client);
// 添加 5 个文件描述符
        fds[i] = accept(sockfd, (struct sockaddr *)&client, &addrlen);
        if (fds[i] > max)
            max = fds[i];
    }

		while (1)
    {
        FD_ZERO(&rset);
        for (i = 0; i < 5; i++)
        {
            FD_SET(fds[i], &rset);
        }

        puts("round again");

				//select 监听全部5个 fd，当有一个或以上 fd 就绪时，就会向下执行
        select(max + 1, &rset, NULL, NULL, NULL);

        for (i = 0; i < 5; i++)
        {
					//判断是哪一个 fd 符合可读状态，执行 read
            if (FD_ISSET(fds[i], &rset))
            {
                memset(buffer, 0, MAXBUF);
                read(fds[i], buffer, MAXBUF);
                puts(buffer);
            }
        }
    }
```

poll 的处理方法与 select 类似，不过它们属于比较早期的方案。现在**广泛使用的方案性能表现更好的 epoll（linux 平台） ， kqueue（ FreeBSD，MacOS）。**



##  信号驱动 I/O ( signal-driven I/O )

信号驱动 I/O模型下，启动操作后，内核会在文件描述符准备就绪时发出信号，我们要根据收到的不同信号，作相应的处理，在等待信号期间进程不阻塞。

实现信号驱动有三个步骤：

1. 设置接受信号的进程，将它和发出信号文件描述符 fd 绑定：`fcntl(fd, F_SETOWN, pid)`
2. 创建信号处理方法，例如 `sa.sa_handler = sigioHandler;`
3. 使用 `sigaction` 函数监听信号（如 SIGIO 信号 ）： `sigaction(SIGIO, &sa, NULL)`

以 write 为例，使用 signal I/O 的关键代码如下：

```c
static volatile sig_atomic_t gotSigio = 0;

static void sigioHandler(int sig){
    gotSigio = 1;
}

int main {
		struct sigaction sa;
		/* Establish handler for "I/O possible" signal */
		sigemptyset(&sa.sa_mask);
		sa.sa_flags = SA_RESTART;
		sa.sa_handler = sigioHandler;
		// On Linux platform
		if (sigaction(SIGIO, &sa, NULL) == -1)
		    printf("sigaction");
		
		/* 设置当前进程作为信号的接受者 */
		if (fcntl(STDOUT_FILENO, F_SETOWN, getpid()) == -1)
		    printf("fcntl(F_SETOWN)");
		
		/* 启动信号功能，设置非阻塞 */
		set_fl(STDOUT_FILENO, O_ASYNC | O_NONBLOCK);

		...
		while (ntowrite > 0)
    {
       ...
        if (gotSigio)
        {
            gotSigio = 0;
            ....
            nwrite = write(STDOUT_FILENO, ptr, ntowrite);
           
					....
        }
    }
}
```

信号驱动 I/O 在应用上并不是很流行，网上一些资料和《The Linux Programming Interface 》（p1355）一书提到，epoll 信号驱动I/O 和 epoll 性能相近，但是有一些劣势，比如，信号处理复杂，因为信号种类繁多，每种都要处理；需要 non-blocking 配合使用。

除此之外，因为无法传参数到 sa_handler 里，只能在里面修改全局变量，使用非常不方便。像上面的例子，只能在接收到信号时修改全局变量 `gotSigio`，然后再去轮询  `gotSigio` 是否变化，十分低效。



##  异步 I/O（asynchronous I/O )

在 UNP 一书的定义里，异步I/O 是指操作启动后，内核**完全处理好这个操作（包括第一、第二阶段）**，再通知进程。

符合这个要求的库在 linux 平台有 `aio` ，不过貌似有一些缺陷，没有太多应用。在 windows 平台，则有比较成熟的 `iocp` 接口。 `iocp` 也是 windows 处理 I/O 操作的最流行的方案。



##  Linux 当前最流行的I/O方案：epoll，kqueue

*inux 平台目前最主要的 I/O 方案是 `epoll` （Linux）和 `kqueue` （BSD等），它们的性能表现最好。

以 `kqueue` 为例，它的处理步骤主要是：

1. 声明一个队列 

```c
int kq = kqueue();
```

2. 声明并初始化要监听的事件 evSet ：

```c
		struct kevent evSet;
    //给 evSet 赋值
    EV_SET(&evSet, localFd, EVFILT_READ, EV_ADD, 0, 0, NULL);
```

3. 把事件添加到队列中（可以添加多个的不同事件）

```c
kevent(kq, &evSet, 1, NULL, 0, NULL)
```

4. 当有一个以上的事件就绪时，读取事件并做相应处理

```c
int nev = kevent(kq, NULL, 0, evList, 32, NULL);

for (int i = 0; i < nev; i++){
		int fd = (int)evList[i].ident;
		if (evList[i].flags & EV_EOF){
			  printf("Disconnect\n");
        close(fd);
    }else if (...){

		}
	.....
}
```

`epoll` 和 `kqueue` 类似，它们的处理方式和上文 `select` 很像，都是把多个文件描述符放到一个监听队列里，当有一个以上事件就绪时，再处理相应的事件。`epoll` 和 `kqueue` 优势是对队列事件的处理更加高效。

目前，包括 **Nodejs 的底层 I/O 库 [libuv](http://docs.libuv.org/en/v1.x/index.html) 和 Golang 的 netpoll ，在 *inux 平台使用的方案都是 epoll 和 kqueue 。**

以 netpoll 为例，I/O 操作最后是交给 epoll 等接口。和直接使用原生 epoll 接口不同的是，golang 是在 goroutine 层面进行这一 I/O 操作，当流程进入 epoll 等接口之后，当前 goroutine 变成阻塞状态，等待 I/O 完成后，再被重新调度。



最后，除了本文已经提到的接口，还有其它的一些 I/O 方案存在，今年以来，linux 平台新近出现的 I/O 接口 `io_uring` 受到不少关注，包括 libuv 等都在尝试引入 `io_uring` 接口，值得继续留意。



文章涉及的代码示例存放在： [io-playground](https://github.com/chasecs/io-playground)



## 参考资料

nonblocking

[http://www.cs.columbia.edu/~jae/4118/L08-adv-io.html](http://www.cs.columbia.edu/~jae/4118/L08-adv-io.html)

select，epoll

[LINUX – IO MULTIPLEXING – SELECT VS POLL VS EPOLL](https://devarea.com/linux-io-multiplexing-select-vs-poll-vs-epoll/#.X0YjxhMzZTY)

[https://eklitzke.org/blocking-io-nonblocking-io-and-epoll](https://eklitzke.org/blocking-io-nonblocking-io-and-epoll)

Blocking vs. non-blocking sockets

[https://www.scottklement.com/rpg/socktut/nonblocking.html

signal I/O

[https://www.man7.org/tlpi/code/online/diff/altio/demo_sigio.c.html](https://www.man7.org/tlpi/code/online/diff/altio/demo_sigio.c.html)

[http://www.cs.fsu.edu/~xyuan/cop5570/lect19_signaldrivenio-1.pptx](http://www.cs.fsu.edu/~xyuan/cop5570/lect19_signaldrivenio-1.pptx)

讨论 signal I/O 为何无法传参到 signal_handler

[https://stackoverflow.com/questions/6970224/providing-passing-argument-to-signal-handler](https://stackoverflow.com/questions/6970224/providing-passing-argument-to-signal-handler)

io_uring

[https://kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf)

[https://lwn.net/Articles/810414/](https://lwn.net/Articles/810414/)

[https://stackoverflow.com/questions/13407542/is-there-really-no-asynchronous-block-i-o-on-linux](https://stackoverflow.com/questions/13407542/is-there-really-no-asynchronous-block-i-o-on-linux)

https://www.scottklement.com/rpg/socktut/nonblocking.html)