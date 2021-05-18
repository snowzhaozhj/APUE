守护进程
--------

守护进程（daemon）是生存期长的一种进程。它们常常在系统引导装入时启动，仅在系统关闭时才终止。它们没有控制终端，在后台运行的。UNIX系统有很多守护进程，它们执行日常事务活动。

守护进程的特征
~~~~~~~~~~~~~~

系统进程依赖于操作系统实现。父进程ID为0的各进程通常是内核进程，它们作为系统引导装入过程的一部分而启动。（init是个例外，它是一个由内核在引导装入时启动的用户层次的命令。）内核进程是特殊的，通常存在于系统的整个生命期中。它们以超级用户特权运行，无控制终端，无命令行。

大多数守护进程都以超级用户（root）特权运行。所有的守护进程都没有控制终端，其终端名设置为问号。内核守护进程以无控制终端方式启动。用户层守护进程缺少控制终端可能是守护进程调用了setsid的结果。大多数用户层守护进程都是进程组的组长进程以及会话的首进程，而且是这些进程组和会话中的唯一进程（rsyslogd是一个例外）。最后，应当注意的是用户层守护进程的父进程是init进程。

编程规则
~~~~~~~~

编写守护进程程序需要遵循一些基本规则：

1. 首先要调用umask将文件模式创建屏蔽字设置为一个已知值（通常是0）。
2. 调用fork，然后使父进程exit。
3. 调用setsid创建一个新会话。使调用进程：（a）成为新会话的首进程，（b）成为一个新进程组的组长进程，（c）没有控制终端。
4. 将当前工作目录更改为根目录。或者，某些守护进程还可能会把当前工作目录更改到某个指定位置，并在此位置进行它们的全部工作。例如，行式打印机假脱机守护进程就可能将其工作目录更改到它们的spool目录上。
5. 关闭不再需要的文件描述符。
6. 某些守护进程打开/dev/null使其具有文件描述符0、1和2，这样，任何一个试图读标准输入、写标准输出或标准错误的库例程都不会产生任何效果。

示例：

.. code:: c

   #include "apue.h"
   #include <syslog.h>
   #include <fcntl.h>
   #include <sys/resource.h>

   void daemonize(const char* cmd) {
       int i, fd0, fd1, fd2;
       pid_t pid;
       struct rlimit r1;
       struct sigaction sa;

       umask(0);
       if (getrlimit(RLIMIT_NOFILE, &r1) < 0) {
           err_quit("%s: can't get file limit", cmd);
       }
       if ((pid = fork()) < 0) {
           err_quit("%s: can't fork", cmd);
       } else if (pid != 0) {
           exit(0);
       }
       setsid();
       sa.sa_handler = SIG_IGN;
       sigemptyset(&sa.sa_mask);
       sa.sa_flags = 0;
       if (sigaction(SIGHUP, &sa, NULL) < 0) {
           err_quit("%s: can't ignore SIGHUP", cmd);
       }
       if ((pid = fork()) < 0) {
           err_quit("%s: can't fork", cmd);
       } else if (pid != 0) {
           exit(0);
       }
       if (chdir("/") < 0) {
           err_quit("%s: can't chdir to /", cmd);
       }
       if (r1.rlim_max == RLIM_INFINITY) {
           r1.rlim_max = 1024;
       }
       for (i = 0; i < r1.rlim_max; i++) {
           close(i);
       }
       fd0 = open("/dev/null", O_RDWR);
       fd1 = dup(0);
       fd2 = dup(0);
       openlog(cmd, LOG_CONS, LOG_DAEMON);
       if (fd0 != 0 || fd1 != 1 || fd2 != 2) {
           syslog(LOG_ERR, "unexpected file descriptors %d %d %d", fd0, fd1, fd2);
           exit(1);
       }
   }

..

   不太明白这个函数到底导致了什么结果，编写的测试程序的结果和想象中有点不同。

出错记录
~~~~~~~~

有三种产生日志消息的方法：

1. 内核例程可以调用log函数。任何一个用户进程都可以通过打开并读/dev/klog设备来读取这些消息。
2. 大多数用户进程（守护进程）调用syslog(3)函数来产生日志消息。这使消息被发送至UNIX域数据报套接字/dev/log。
3. 无论一个用户进程是在此主机上，还是在通过TCP/IP网络连接到此主机的其他主机上，都可将日志消息发向UDP端口514。注意，syslog函数从不产生这些UDP数据报，它们要求产生此日志消息的进程进行显式的网络编程。

通常，syslogd守护进程读取所有3种格式的日志消息。此守护进程在启动时读一个配置文件，其文件名一般为/etc/syslog.conf，该文件决定了不同种类的消息应送向何处。

.. code:: c

   #include <syslog.h>

   void openlog(const char* ident, int option, int facility);
   void syslog(int priority, const char* format, ...);
   void closelog(void);
   int setlogmask(int maskpri);
   // 返回之前的日志记录优先级屏蔽字

调用openlog是可选择的。如果不调用openlog，则在第一次调用syslog时，自动调用openlog。

调用
closelog也是可选择的，因为它只是关闭曾被用于与syslogd守护进程进行通信的描述符。

调用openlog
使我们可以指定一个ident，以后，此ident将被加至每则日志消息中。ident一般是程序的名称（如cron、inetd）。

option参数是指定各种选项的位屏蔽：

|image0|

facility参数：

priority参数是facility和level的组合。

level的可能取值(优先级从高到低排序)：

如果不调用openlog，或者以facility为0来调用它，那么在调用syslog时，可将facility作为priority参数的一个部分进行说明。

setlogmask函数用于设置进程的记录优先级屏蔽字。它返回调用它之前的屏蔽字。

当设置了记录优先级屏蔽字时，各条消息除非已在记录优先级屏蔽字中进行了设置，否则将不被记录。

   注意，试图将记录优先级屏蔽字设置为0并不会有什么作用。

很多平台还提供了syslog的一种变体：

.. code:: c

   #include <syslog.h>
   #include <stdarg.h>

   void vsyslog(int priority const char* format, va_list arg);

单实例守护进程
~~~~~~~~~~~~~~

为了正常运作，某些守护进程会实现为，在任一时刻只运行该守护进程的一个副本。

文件和记录锁机制为一种方法提供了基础，该方法保证一个守护进程只有一个副本在运行。如果每一个守护进程创建一个有固定名字的文件，并在该文件的整体上加一把写锁，那么只允许创建一把这样的写锁。在此之后创建写锁的尝试都会失败，这向后续守护进程副本指明已有一个副本正在运行。

文件和记录锁提供了一种方便的互斥机制。如果守护进程在一个文件的整体上得到一把写锁，那么在该守护进程终止时，这把锁将被自动删除。

示例程序：

.. code:: c

   #include <unistd.h>
   #include <stdlib.h>
   #include <fcntl.h>
   #include <syslog.h>
   #include <string.h>
   #include <errno.h>
   #include <stdio.h>
   #include <sys/stat.h>

   #define LOCKFILE "/var/run/daemon.pid"
   #define LOCKMODE (S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH)

   extern int lockfile(int);

   int already_running() {
       int fd;
       char buf[16];
       fd = open(LOCKFILE, O_RDWR|O_CREAT, LOCKMODE);
       if (fd < 0) {
           syslog(LOG_ERR, "can't open %s: %s", LOCKFILE, strerror(errno));
       }
       if (lockfile(fd) < 0) {
           if (errno == EACCES || errno == EAGAIN) {
               close(fd);
               return 1;
           }
           syslog(LOG_ERR, "can't lock %s: %s", LOCKFILE, strerror(errno));
           exit(1);
       }
       ftruncate(fd, 0);
       sprintf(buf, "%ld", (long)getpid());
       write(fd, buf, strlen(buf) + 1);
       return 0;
   }

..

   不太懂lockfile函数在哪，没找到:sob:。

守护进程的惯例
~~~~~~~~~~~~~~

在UNIX系统中，守护进程遵循下列通用惯例：

-  若守护进程使用锁文件，那么该文件通常存储在/var/run目录中。锁文件的名字通常是name.pid，其中，name是该守护进程或服务的名字。
-  若守护进程支持配置选项，那么配置文件通常存放在/etc目录中。配置文件的名字通常是name.conf。
-  守护进程可用命令行启动，但通常它们是由系统初始化脚本之一（/etc/rc*或/etc/init.d/*）启动的。如果在守护进程终止时，应当自动地重新启动它（我们可在/etc/inittab中为该守护进程包括respawn记录项，这样，init就将重新启动该守护进程）。
-  若一个守护进程有一个配置文件，那么当该守护进程启动时会读该文件，但在此之后一般就不会再查看它。若某个管理员更改了配置文件，那么该守护进程可能需要被停止，然后再启动，以使配置文件的更改生效。为避免此种麻烦，某些守护进程将捕捉SIGHUP信号，当它们接收到该信号时，重新读配置文件。

客户进程-服务器进程模型
~~~~~~~~~~~~~~~~~~~~~~~

守护进程常常用作服务器进程。

一般而言，服务器进程等待客户进程与其联系，提出某种类型的服务要求。

.. |image0| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00303.jpg
