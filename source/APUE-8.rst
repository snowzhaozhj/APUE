进程控制
--------

进程标识
~~~~~~~~

每个进程都有一个非负整型表示的唯一进程ID。

虽然是唯一的，但是进程ID是可复用的。当一个进程终止后，其进程ID就成为复用的候选者。

系统中有一些专用进程，具体细节随实现而不同：

-  ID为0的进程通常是调度进程，常常被称为\ **交换进程**\ （swapper）。该进程是内核的一部分，它并不执行任何磁盘上的程序，因此也被称为系统进程。

-  进程ID为1的进程通常是\ **init进程**\ ，在自举过程结束时由内核调用。此进程负责在自举内核后启动一个UNIX系统。init进程决不会终止。它是一个普通的用户进程（与交换进程不同，它不是内核中的系统进程），但是它以超级用户特权运行。

-  在某些UNIX的虚拟存储器实现中，进程ID为2的进程是\ **页守护进程**\ （page
   daemon），此进程负责支持虚拟存储器系统的分页操作。

可以通过以下函数来获取进程的一些标识符：

.. code:: c

   #include <unistd.h>

   pid_t getpid(void);     // 返回调用进程的进程ID
   pid_t getppid(void);    // 返回调用进程的父进程ID
   uid_t getuid(void);     // 返回调用进程的实际用户ID
   uid_t geteuid(void);    // 返回调用进程的有效用户ID
   git_t getgid(void);     // 返回调用进程的实际组ID
   gid_t getegid(void);    // 返回调用进程的有效组ID

   // 这些函数都没有出错返回

fork
~~~~

一个现有的进程可以调用fork函数来创建一个新的线程：

.. code:: c

   #include <unistd.h>

   pid_t fork(void);
   // 子进程返回0，父进程返回子进程ID，出错返回-1

子进程是父进程的副本，获得父进程数据空间、堆和栈的副本。

父进程和子进程共享正文段。

   由于在fork之后经常跟随着exec，所以现在的很多实现并不执行一个父进程数据段、栈和堆的完全副本。作为替代，使用了写时复制（Copy-On-Write，COW）技术。这些区域由父进程和子进程共享，而且内核将它们的访问权限改变为只读。如果父进程和子进程中的任一个试图修改这些区域，则内核只为修改区域的那块内存制作一个副本，通常是虚拟存储系统中的一“页”。

fork函数的简单演示：

.. code:: c

   #include "apue.h"

   int globvar = 6;
   char buf[] = "a write to stdout\n";

   int main() {
       int var;
       pid_t pid;
       var = 88;
       if (write(STDOUT_FILENO, buf, sizeof(buf) - 1) != sizeof(buf) - 1) {
           err_sys("write error!");
       }
       printf("before fork\n"); // don't flush stdout
       if ((pid = fork()) < 0) {
           err_sys("fork error!");
       } else if (pid == 0) { // child process
           globvar++;
           var++;
       } else { // parent process
           sleep(2);
       }
       printf("pid = %ld, glob = %d, var = %d\n", (long)getpid(), globvar, var);
       return 0;
   }

直接输出结果：

::

   a write to stdout
   before fork
   pid = 25767, glob = 7, var = 89
   pid = 25766, glob = 6, var = 88

重定向到某个文件后的输出结果：

::

   a write to stdout
   before fork
   pid = 25790, glob = 7, var = 89
   before fork
   pid = 25789, glob = 6, var = 88

..

   重定向会使得标准输出从“行缓冲”变为“全缓冲”，进而导致上述结果的不同。

fork的一个特性是父进程的所有打开文件描述符都被复制到子进程中。父进程和子进程每个相同的打开描述符共享一个文件表项，就好像执行了dup函数一样。

因此在重定向父进程的标准输出时，子进程的标准输出也被重定向。

除了打开文件之外，父进程的很多其他属性也由子进程继承，包括：

-  实际用户ID、实际组ID、有效用户ID、有效组ID、附属组ID、进程组ID、会话ID；
-  控制终端；
-  设置用户ID标志和设置组ID标志；
-  当前工作目录、根目录；
-  文件模式创建屏蔽字；
-  信号屏蔽和安排；
-  对任一打开文件描述符的执行时关闭（close-on-exec）标志；
-  环境、连接的共享存储段、存储映像、资源限制。

父进程和子进程之间的区别：

-  fork的返回值不同；
-  进程ID不同，父进程ID不同；
-  子进程的tms_utime、tms_stime、tms_cutime和tms_ustime的值设置为0；
-  子进程不继承父进程设置的文件锁；
-  子进程的未处理闹钟被清除；
-  子进程的未处理信号集设置为空集。

vfork
~~~~~

vfork函数用于创建一个新进程，而该新进程的目的是exec一个新程序。

vfork与fork一样都创建一个子进程，但是它并不将父进程的地址空间完全复制到子进程中。

在子进程调用exec或exit之前，它\ **在父进程的空间中运行**\ 。

vfork保证子进程先运行，在它调用exec或exit之后父进程才可能被调度运行，当子进程调用这两个函数中的任意一个时，父进程会恢复运行。

exit
~~~~

进程有5种正常终止及3种异常终止方式。

5种正常终止方式：

1. 在main函数内执行return语句，这等效于调用exit。

2. 调用exit函数。其操作包括调用各终止处理程序（终止处理程序在调用atexit函数时登记），然后关闭所有标准I/O流等。

3. 调用\ ``_exit``\ 或\ ``_Exit``\ 函数。

4. 进程的最后一个线程在其启动例程中执行return语句。但是，该线程的返回值不用作进程的返回值。当最后一个线程从其启动例程返回时，该进程以终止状态0返回。

5. 进程的最后一个线程调用pthread_exit函数。进程终止状态总是0，与传送给pthread_exit的参数无关。

3种异常终止方式：

6. 调用abort。它产生SIGABRT信号。
7. 当进程接收到某些信号时。信号可由进程自身（如调用abort函数）、其他进程或内核产生。
8. 最后一个线程对“取消”（cancellation）请求作出响应。

对于父进程已经终止的所有进程，它们的父进程都改变为init进程。这个过程被称为\ **init进程收养**\ 。

一个已经终止、但是其父进程尚未对其进行善后处理（获取终止子进程的有关信息、释放它仍占用的资源）的进程被称为\ **僵死进程**\ （zombie）。ps(1)命令将僵死进程的状态打印为Z。

wait和waitpid
~~~~~~~~~~~~~

当一个进程正常或异常终止时，内核就向其父进程发送 SIGCHLD
信号。因为子进程终止是个异步事件（这可以在父进程运行的任何时候发生），所以这种信号也是内核向父进程发的异步通知。

调用wait和waitpid的可能影响：

-  如果其所有子进程都还在运行，则阻塞。

-  如果一个子进程已终止，正等待父进程获取其终止状态，则取得该子进程的终止状态立即返回。

-  如果它没有任何子进程，则立即出错返回。

.. code:: c

   #include <sys/wait.h>

   pid_t wait(int *statloc);
   pid_t waitpid(pid_t pid, int *statloc, int options);
   // 成功返回进程ID，出错返回0或-1

statloc是一个整型指针。如果statloc不是一个空指针，则终止进程的终止状态就存放在它所指向的单元内。如果不关心终止状态，则可将该参数指定为空指针。

可以通过以下的宏对statloc进行测试（其中的status=*statloc）：

|image0|

waitpid函数中pid参数的作用：

-  pid == −1：等待任一子进程。此种情况下，waitpid与wait等效。

-  pid > 0：等待进程ID与pid相等的子进程。

-  pid == 0：等待组ID等于调用进程组ID的任一子进程。

-  pid < −1：等待组ID等于pid绝对值的任一子进程。

waitpid的options参数：

|image1|

对于wait，其唯一的出错是调用进程没有子进程。但是对于waitpid，如果指定的进程或进程组不存在，或者参数pid指定的进程不是调用进程的子进程，都可能出错。

一个有趣的例子，通过fork两次，让父进程无需等待子进程终止，子进程无需处于僵死状态直到父进程终止。

.. code:: c

   #include "apue.h"
   #include <sys/wait.h>

   int main() {
       pid_t pid;
       if ((pid = fork()) < 0) {
           err_sys("fork error");
       } else if (pid == 0) {
           if ((pid = fork()) < 0) {
               err_sys("fork error");
           } else if (pid > 0) {
               exit(0);
           }
           sleep(2);
           // when cur process's parent called exit(0),
           // cur process will be adopted by the init process
           printf("second child, parent pid = %ld\n", (long)getppid());
           exit(0);
       }
       if (waitpid(pid, NULL, 0) != pid) {
           err_sys("waitpid error");
       }
       exit(0);
   }

waitid
~~~~~~

Single UNIX
Specification包括了另一个取得进程终止状态的函数waitid，此函数类似于waitpid，但提供了更多的灵活性。

.. code:: c

   #include <sys/wait.h>

   int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
   // 成功返回0，出错返回-1

该函数支持的idtype和options如下：

|image2|

infop参数是指向siginfo结构的指针。该结构包含了造成子进程状态改变有关信号的详细信息。

wait3和wait4
~~~~~~~~~~~~

wait3和wait4是从BSD分支沿袭下来的，他们提供的功能比POSIX.1函数wait,
waitpid, waitid要多一个。

.. code:: c

   #include <sys/types.h>
   #include <sys/wait.h>
   #include <sys/time.h>
   #include <sys/resource.h>

   pid_t wait3(int *statloc, int options, struct rusage* rusage);
   pid_t wait4(pid_t pid, int *statloc, int options, struct rusage* rusage);
   // 成功返回进程ID，出错返回-1

exec
~~~~

当进程调用exec函数时，该进程执行的程序完全替换为新程序，而新程序则从其main函数开始执行。

调用exec并不创建新进程，exec只是用磁盘上的一个新程序替换了当前进程的正文段、数据段、堆段和栈段。

.. code:: c

   #include <unistd.h>

   int execl(const char* pathname, const char* arg0, ... /* (char*)0 */);
   int execv(const char* pathname, char* const argv[]);
   int execle(const char* pathname, const char* arg0, ... /* (char*)0 char* const envp[] */);
   int execve(const char* pathname, char* const argv[], char* const envp[]);
   int execlp(const char* pathname, const char* arg0, ... /* (char*)0 */);
   int execvp(const char* pathname, char* const argv[]);
   int fexecve(int fd, char* const argv[], char* const envp[]);
   // 成功不返回，出错返回-1

带有p的函数：

-  如果filename中包含/，则就将其视为路径名；

-  否则就按PATH环境变量，在它所指定的各目录中搜寻可执行文件。

..

   如果execlp或execvp使用路径前缀中的一个找到了一个可执行文件，但是该文件不是由连接编辑器产生的机器可执行文件，则就认为该文件是一个shell脚本，于是试着调用/bin/sh，并以该filename作为shell的输入。

l代表list，v代表vector。

以e结尾的函数可以传递一个指向环境字符串指针数组的指针。其他4个函数则使用调用进程中的environ变量为新程序复制现有的环境。

在exec前后实际用户ID和实际组ID保持不变，而有效ID是否改变则取决于所执行程序文件的设置用户ID位和设置组ID位是否设置。如果新程序的设置用户ID位已设置，则有效用户ID变成程序文件所有者的ID；否则有效用户ID不变。对组ID的处理方式与此相同。

更改用户ID和更改组ID
~~~~~~~~~~~~~~~~~~~~

可以通过setuid和setgid来设置用户ID和组ID：

.. code:: c

   #include <unistd.h>

   int setuid(uid_t uid);
   int setgid(gid_t gid);
   // 成功返回0，出错返回-1

其中的更改规则可参考下图（还包括exec函数的情况）：

|image3|

可以通过setreuid和setregid来交换实际用户ID和有效用户ID的值：

.. code:: c

   #include <unistd.h>

   int setreuid(uid_t ruid, uid_t euid);
   int setregid(gid_t rgid, gid_t egid);
   // 成功返回0，失败返回-1

可以通过seteuid和setegid来设置有效ID：

.. code:: c

   #include <unistd.h>

   int seteuid(uid_t uid);
   int setegid(gid_t gid);
   // 成功返回0，出错返回-1

解释器文件
~~~~~~~~~~

解释器文件（interpreter file）是文本文件，其起始行的形式是：

.. code:: sh

   #! pathname [ optional-argument ]

个人理解：如果指定了起始行的话，就会在原来的argv数组前插入起始行的参数。

system
~~~~~~

可以使用system函数在程序中执行一个命令行命令：

.. code:: c

   #include <stdlib.h>

   int system(const char* cmdstring);

如果cmdstring是一个空指针，则仅当命令处理程序可用时，system返回非0值。

system在其实现中调用了fork、exec和waitpid，因此有3种返回值。

1. fork失败或者waitpid返回除EINTR之外的出错，则system返回−1，并且设置errno。

2. exec失败（表示不能执行shell），则其返回值如同shell执行了exit(127)一样。

3. 如果所有3个函数（fork、exec和waitpid）都成功，那么system的返回值是shell的终止状态。

设置用户ID或设置组ID程序决不应调用system函数。

进程会计
~~~~~~~~

超级用户执行一个带路径名参数的accton命令启用会计处理。会计记录写到指定的文件中，在FreeBSD和Mac
OS
X中，该文件通常是/var/account/acct；在Linux中，该文件是/var/account/pacct；在Solaris中，该文件是/var/adm/pacct。执行不带任何参数的accton命令则停止会计处理。

典型的会计记录包含总量较小的二进制数据，一般包括命令名、所使用的CPU时间总量、用户ID和组ID、启动时间等。

会计记录所需的各个数据（各CPU时间、传输的字符数等）都由内核保存在进程表中，并在一个新进程被创建时初始化（如fork之后在子进程中）。进程终止时写一个会计记录。

会计记录的注意事项：

1. 我们不能获取永远不终止的进程的会计记录。像init这样的进程在系统生命周期中一直在运行，并不产生会计记录。这也同样适合于内核守护进程，它们通常不会终止。
2. 在会计文件中记录的顺序对应于进程终止的顺序，而不是它们启动的顺序。
3. 会计记录对应于进程而不是程序。

用户标识
~~~~~~~~

可以使用getlogin函数来获取用户登录名：

.. code:: c

   #include <unistd.h>

   char* getlogin(void);
   // 成功返回指向登录名字符串的指针，出错返回NULL

如果调用此函数的进程没有连接到用户登录时所用的终端，则函数会失败。通常称这些进程为守护进程（daemon）。

给出了登录名，就可用getpwnam在口令文件中查找用户的相应记录，从而确定其登录shell等。

进程调度
~~~~~~~~

NZERO是系统默认的友好值。

进程可以通过nice函数获取或更改它的nice值。使用这个函数，进程只能影响自己的nice值，不能影响任何其他进程的nice值。

.. code:: c

   #include <unistd.h>

   int nice(int incr);
   // 成功返回新的友好值，出错返回-1

ncr参数被增加到调用进程的nice值上。如果incr太大，系统直接把它降到最大合法值。如果incr太小，系统会无声息地把它提高到最小合法值。

可以使用getpriority函数来获取友好值：

.. code:: c

   #include <sys/resource.h>

   int getpriority(int which, id_t who);
   // 成功返回-NZERO~NZEROR-1之间的友好值，出错返回-1

which参数控制who参数是如何解释的：

-  PRIO_PROCESS：进程；
-  PRIO_PGRP：进程组；
-  PRIO_USER：用户ID。

如果who参数为0，表示调用进程、进程组或者用户。

当which设为PRIO_USER并且who为0时，使用调用进程的实际用户ID。

如果which参数作用于多个进程，则返回所有作用进程中优先级最高的（最小的nice值）。

可以使用setpriority函数来为进程、进程组和属于特定用户ID的所有进程设置优先级：

.. code:: c

   #include <sys/resource.h>

   int setpriority(int which, id_t who, int value);
   // 成功返回0，出错返回-1

进程时间
~~~~~~~~

任意进程都可以通过times函数来获取进程时间：

.. code:: c

   #include <sys/times.h>

   clock_t times(struct tms* buf);
   // 成功返回流逝的墙上时钟时间，出错返回-1

其中\ ``struct tms``\ 的结构为：

.. code:: c

   struct tms {
       clock_t tms_utime;  // user CPU time
       clock_t tms_stime;  // system CPU time
       clock_t tms_cutime; // user CPU time, terminated children
       clock_t tms_cstime; // system CPU time, terminated children
   };

该结构中两个针对子进程的字段包含了此进程用wait函数族已等待到的各子进程的值。

.. |image0| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00152.jpg
.. |image1| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00156.jpg
.. |image2| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00159.jpg
.. |image3| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00168.jpg
