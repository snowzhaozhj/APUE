进程关系
--------

终端登录
~~~~~~~~

Unix系统传统的用户身份验证
^^^^^^^^^^^^^^^^^^^^^^^^^^

当系统自举时，内核创建init进程。init进程使系统进入多用户模式。init进程读取文件/etc/ttys，对每一个允许登录的终端设备，init调用一次fork，它所生成的子进程则exec
getty程序。

getty对终端设备调用open函数，以读、写方式将终端打开。

当用户键入了用户名后，getty的工作就完成了。然后它以类似于下列的方式调用login程序：

.. code:: c

   execle("/bin/login", "login", "-p", username, (char *)0, envp);

现代UNIX系统的多身份验证
~~~~~~~~~~~~~~~~~~~~~~~~

FreeBSD、Linux、Mac OS X以及Solaris都支持被称为 PAM（Pluggable
Authentication Modules，可插入的身份验证模块）的更加灵活的方案。PAM
允许管理人员配置使用何种身份验证方法来访问那些使用PAM库编写的服务。

如果用户正确登录，login会完成如下工作：

-  将当前工作目录更改为该用户的起始目录（chdir）。

-  调用chown更改该终端的所有权，使登录用户成为它的所有者。

-  将对该终端设备的访问权限改变成“用户读和写”。

-  调用setgid及initgroups设置进程的组ID。

-  用login得到的所有信息初始化环境：起始目录（HOME）、shell（SHELL）、用户名（USER和LOGNAME）以及一个系统默认路径（PATH）。

-  login进程更改为登录用户的用户ID（setuid）并调用该用户的登录shell，其方式类似于：

   .. code:: c

      execl("/bin/sh", "-sh", (char *)0);

   ..

      其中argv[0]的第一个字符负号是一个标志，表示该shell被作为登录shell调用。

然后登录shell读取启动文件(如\ ``.bashrc``\ 等)，当执行完启动文件后，用户最后得到
shell提示符，并能键入命令。

网络登录
~~~~~~~~

在网络登录情况下，login仅仅是一种可用的服务，这与其他网络服务（如FTP或SMTP）的性质相同。

为使同一个软件既能处理终端登录，又能处理网络登录，系统使用了一种称为伪终端（pseudo
terminal）的软件驱动程序，它仿真串行终端的运行行为，并将终端操作映射为网络操作，反之亦然。

BSD网络登录：

作为系统启动的一部分，init调用一个shell，使其执行shell脚本/etc/rc。由此shell脚本启动一个守护进程inetd。一旦此shell脚本终止，inetd的父进程就变成init。inetd等待TCP/IP连接请求到达主机，而当一个连接请求到达时，它执行一次fork，然后生成的子进程exec适当的程序。

   inetd有时被称为因特网超级服务器，它等待大多数网络连接。

其他系统的登录也大致相同。

进程组
~~~~~~

每个进程除了有一进程ID之外，还属于一个进程组。

进程组是一个或多个进程的集合。通常，它们是在同一作业中结合起来的，同一进程组中的各进程接收来自同一终端的各种信号。每个进程组有一个唯一的进程组ID。

可以使用getpgrp函数来获得调用进程的进程组ID：

.. code:: c

   #include <unistd.h>

   pid_t getpgrp(void);
   // 返回调用进程的进程组ID

-  每个进程组有一个组长进程。\ **组长进程的进程组ID等于其进程ID。**

-  进程组组长可以创建一个进程组、创建该组中的进程，然后终止。

-  只要在某个进程组中有一个进程存在，则该进程组就存在，这与其组长进程是否终止无关。

-  从进程组创建开始到其中最后一个进程离开为止的时间区间称为进程组的生命期。

-  某个进程组中的最后一个进程可以终止，也可以转移到另一个进程组。

进程可以调用setpgid来加入一个现有的进程组或者创建一个新的进程组：

.. code:: c

   #include <unistd.h>

   int setpgid(pid_t pid, pid_t pgid);
   // 成功返回0，出错返回-1

setpgid函数将pid进程的进程组ID设置为pgid。

-  如果这两个参数相等，则由pid指定的进程变成进程组组长。
-  如果pid是0，则使用调用者的进程ID。
-  如果pgid是0，则由pid指定的进程ID用作进程组ID。

一个进程只能为它自己或它的子进程设置进程组ID。在它的子进程调用了exec后，它就不再更改该子进程的进程组ID。

会话
~~~~

会话（session）是一个或多个进程组的集合。

通常是由shell的管道将几个进程编成一组的。

进程可以调用setsid函数来创建一个新会话：

.. code:: c

   #include <unistd.h>

   pid_t setsid(void);
   // 成功返回进程组ID，出错返回-1

如果调用此函数的进程不是一个进程组的组长，则此函数创建一个新会话：

1. 该进程变成新会话的会话首进程（session
   leader，会话首进程是创建该会话的进程）。

2. 该进程成为一个新进程组的组长进程。新进程组ID是该调用进程的进程ID。

3. 该进程没有控制终端。如果在调用setsid之前该进程有一个控制终端，那么这种联系也被切断。

如果该调用进程已经是一个进程组的组长，则此函数返回出错。

可以调用getsid函数来获取会话首进程的进程组ID：

.. code:: c

   #include <unistd.h>

   pid_t getsid(pid_t pid);
   // 成功返回会话首进程的进程组ID，出错返回-1

如若pid是0，getsid返回调用进程的会话首进程的进程组ID。

   出于安全方面的考虑，一些实现有如下限制：如若pid并不属于调用者所在的会话，那么调用进程就不能得到该会话首进程的进程组ID。

控制终端
~~~~~~~~

会话和进程组的其他特性：

-  一个会话可以有一个\ **控制终端**\ 。通常是终端设备（终端登录）或伪终端设备（网络登录）。

-  建立与控制终端连接的会话首进程被称为\ **控制进程**\ 。

-  一个会话中的几个进程组可被分成\ **一个前台进程组**\ 以及\ **一个或多个后台进程组**\ 。

-  如果一个会话有一个控制终端，则它有一个前台进程组，其他进程组为后台进程组。

-  无论何时键入终端的中断键（常常是Delete或Ctrl+C），都会将中断信号发送至前台进程组的所有进程。

-  无论何时键入终端的退出键（常常是Ctrl+），都会将退出信号发送至前台进程组的所有进程。

-  如果终端接口检测到调制解调器（或网络）已经断开连接，则将挂断信号发送至控制进程。

tcgetpgrp, tcsetpgrp, tcgetsid
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以通过tcgetpgrp和tcsetpgrp来控制前台进程组：

.. code:: c

   #include <unistd.h>

   pid_t tcgetpgrp(int fd);
   // 成功返回前台进程组ID，出错返回-1
   int tcsetpgrp(int fd, pid_t pgrpid);
   // 成功返回0，出错返回-1

如果进程有一个控制终端，则该进程可以调用tcsetpgrp将前台进程组ID设置为pgrpid。

-  pgrpid值应当是在同一会话中的一个进程组的ID。
-  fd必须引用该会话的控制终端。

可以通过tcgetsid函数来获得会话首进程的进程组ID：

.. code:: c

   #include <termios.h>

   pid_t tcgetsid(int fd);
   // 成功返回会话首进程进程组ID，出错返回-1

作业控制
~~~~~~~~

作业控制要求以下3种形式的支持。

1. 支持作业控制的shell。

2. 内核中的终端驱动程序必须支持作业控制。

3. 内核必须提供对某些作业控制信号的支持。

有3个特殊字符可使终端驱动程序产生信号，并将它们发送至前台进程组：

-  中断字符（一般采用Delete或Ctrl+C）产生SIGINT；

-  退出字符（一般采用Ctrl+）产生SIGQUIT；

-  挂起字符（一般采用Ctrl+Z）产生SIGTSTP。

只有前台作业接收终端输入。如果后台作业试图读终端，这并不是一个错误，但是终端驱动程序将检测这种情况，并且向后台作业发送一个特定信号SIGTTIN。该信号通常会停止此后台作业，而shell则向有关用户发出这种情况的通知，然后用户就可用shell命令将此作业转为前台作业运行，于是它就可读终端。

我们可以通过stty(1)命令来控制后台作业是否允许输出到控制终端。当被禁止时，会向作业发送SIGTTOU信号，使其阻塞(类似SIGTTIN)。

shell执行程序
~~~~~~~~~~~~~

这一部分我在自己电脑上测试的结果和书上的结果不太一样，自己也没搞得很明白，暂时不记录笔记。

孤儿进程组
~~~~~~~~~~

POSIX.1将孤儿进程组（orphaned process
group）定义为：该组中每个成员的父进程要么是该组的一个成员，要么不是该组所属会话的成员。

一个进程组不是孤儿进程组的条件是：该组中有一个进程，其父进程在属于同一会话的另一个组中。

如果进程组不是孤儿进程组，那么在属于同一会话的另一个组中的父进程就有机会重新启动该组中停止的进程。

一个示例：

.. code:: c

   #include "apue.h"
   #include <errno.h>

   static void sig_hup(int signo) {
       printf("SIGHUP received, pid = %ld\n", (long) getpid());
   }

   static void pr_ids(char *name) {
       printf("%s: pid = %ld, ppid = %ld, pgrp = %ld, tpgrp = %ld\n", name, (long) getpid(), (long) getppid(),
              (long) getpgrp(), (long) tcgetpgrp(STDIN_FILENO));
       fflush(stdout);
   }

   int main() {
       char c;
       pid_t pid;
       pr_ids("parent");
       if ((pid = fork()) < 0) {
           err_sys("fork error");
       } else if (pid > 0) {
           sleep(5);
       } else {
           pr_ids("child");
           signal(SIGHUP, sig_hup);
           kill(getpid(), SIGTSTP);
           pr_ids("child");
           if (read(STDIN_FILENO, &c, 1) != 1) {
               printf("read error %d on controlling TTY\n", errno);
           }
           exit(0);
       }
   }

..

   我使用clion运行时，结果和书上类似，但是手动通过gcc编译，然后在wsl中运行的结果和书上不太一样，目前没搞清楚怎么回事。

FreeBSD实现
~~~~~~~~~~~

每个会话都分配一个\ ``session``\ 结构：

-  s_count是该会话中的进程组数。当此计数器减至0时，则可释放此结构。

-  s_leader是指向会话首进程proc结构的指针。

-  s_ttyvp是指向控制终端vnode结构的指针。

-  s_ttyp是指向控制终端tty结构的指针。

-  s_sid是会话ID。

在调用setsid时，在内核中分配一个新的session结构：

-  s_count设置为1。
-  s_leader设置为调用进程proc结构的指针。
-  s_sid设置为进程ID。
-  因为新会话没有控制终端，所以s_ttyvp和s_ttyp设置为空指针。

每个终端设备和每个伪终端设备均在内核中分配\ ``tty``\ 结构：

-  t_session指向将此终端作为控制终端的session结构。终端在失去载波信号时使用此指针将挂起信号发送给会话首进程。

-  t_pgrp指向前台进程组的pgrp结构。终端驱动程序用此字段将信号发送给前台进程组。由输入特殊字符（中断、退出和挂起）而产生的3个信号被发送至前台进程组。

-  t_termios是包含所有这些特殊字符和与该终端有关信息（如波特率、回显打开或关闭等）的结构。

-  t_winsize是包含终端窗口当前大小的winsize型结构。当终端窗口大小改变时，信号SIGWINCH被发送至前台进程组。

为了找到特定会话的前台进程组，内核从session结构开始，然后用s_ttyp得到控制终端的tty结构，再用t_pgrp得到前台进程组的pgrp结构。

``pgrp``\ 结构包含一个特定进程组的信息：

-  pg_id是进程组ID。

-  pg_session指向此进程组所属会话的session结构。

-  pg_members是指向此进程组proc结构表的指针，该proc结构代表进程组的成员。proc结构中p_pglist结构是双向链表，指向该组中的下一个进程和上一个进程。直到遇到进程组中的最后一个进程，它的proc结构中p_pglist结构为空指针。

``proc``\ 结构包含一个进程的所有信息：

-  p_pid包含进程ID。

-  p_pptr是指向父进程proc结构的指针。

-  p_pgrp指向本进程所属的进程组的pgrp结构的指针。

-  p_pglist是一个结构，其中包含两个指针，分别指向进程组中上一个和下一个进程。
