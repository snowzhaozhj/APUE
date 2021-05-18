信号
----

信号概念
~~~~~~~~

信号是软件中断。每个信号都有一个名字，这些名字都以“SIG”开头。

在头文件<signal.h>中，信号名都被定义为正整数常量（信号编号）。不存在编号为0的信号。

产生信号的一些途径：

-  按某些终端键会引发终端产生信号。

-  硬件异常产生信号：除数为0、无效的内存引用等。这些条件通常由硬件检测到，并通知内核。然后内核为该条件发生时正在运行的进程产生适当的信号。

-  进程调用kill(2)函数可将任意信号发送给另一个进程或进程组。权限要求：接收信号进程和发送信号进程的所有者必须相同，或发送信号进程的所有者必须是超级用户。

-  在终端使用kill(1)命令将信号发送给其他进程。此命令是kill函数的接口

-  当检测到某种软件条件已经发生，并应将其通知有关进程时也产生信号。例如
   SIGURG（在网络连接上传来带外的数据）、SIGPIPE（在管道的读进程已终止后，一个进程写此管道）以及
   SIGALRM（进程所设置的定时器已经超时）。

信号的处理方式：

-  忽略信号。

-  捕捉信号。

-  执行系统默认动作。对大多数信号的系统默认动作是终止该进程。

..

   SIGKILL和SIGSTOP信号不能被忽略和捕捉。

   这两种信号不能被忽略的原因是：它们向内核和超级用户提供了使进程终止或停止的可靠方法。另外，如果忽略某些由硬件异常产生的信号（如非法内存引用或除以0），则进程的运行行为是未定义的。

信号的详细介绍请参考APUE第三版P252-P256。

signal
~~~~~~

.. code:: c

   #include <signal.h>

   void (*signal(int signo, void (*func)(int)))(int);
   // 成功返回以前的信号处理程序，出错返回SIG_ERR

   // 可以用typedef简化函数原型
   typedef void (*Sigfunc)(int);
   Sigfunc* signal(int, Sigfunc*);

-  signo参数是信号名。

-  func的值是常量SIG_IGN、常量SIG_DFL或信号处理函数的地址。

   -  如果指定SIG_IGN，则向内核表示忽略此信号。
   -  如果指定SIG_DFL，则表示接到此信号后的动作是系统默认动作。
   -  当指定函数地址时，则在信号发生时，调用该函数，我们称这种处理为捕捉该信号，称此函数为信号处理程序（signal
      handler）或信号捕捉函数（signal-catching function）。

一个简单的示例：

.. code:: c

   #include "apue.h"

   static void sig_usr(int);

   int main() {
       if (signal(SIGUSR1, sig_usr) == SIG_ERR) {
           err_sys("can't catch SIGUSR1");
       }
       if (signal(SIGUSR2, sig_usr) == SIG_ERR) {
           err_sys("can't catch SIGUSR2");
       }
       for (;;) {
           pause();
       }
   }

   static void sig_usr(int signo) {
       if (signo == SIGUSR1) {
           printf("received SIGUSR1\n");
       } else if (signo == SIGUSR2) {
           printf("received SIGUSR2\n");
       } else {
           err_dump("received signal %d\n", signo);
       }
   }

中断的系统调用
~~~~~~~~~~~~~~

早期UNIX系统的一个特性是：如果进程在执行一个低速系统调用而阻塞期间捕捉到一个信号，则该系统调用就被中断不再继续执行。该系统调用返回出错，其errno设置为EINTR。

   系统调用分成两类：低速系统调用和其他系统调用。低速系统调用是可能会使进程永远阻塞的一类系统调用。

为了帮助应用程序使其不必处理被中断的系统调用，4.2BSD引进了某些被中断系统调用的自动重启动。

自动重启动的系统调用包括：ioctl、read、readv、write、writev、wait
和waitpid。

前5个函数只有对低速设备进行操作时才会被信号中断。而wait和waitpid在捕捉到信号时总是被中断。

可重入函数
~~~~~~~~~~

SUS说明了在信号处理程序中保证调用安全的函数。这些函数是可重入的并被称为是异步信号安全的（async-signal
safe）。

不可重入的可能情况：

-  已知它们使用静态数据结构；
-  它们调用 malloc 或free；
-  它们是标准I/O函数。

SIGCLD语义
~~~~~~~~~~

在linux里，SIGCLD与SIGCHLD相同。

对于SIGCLD的早期处理方式：

-  如果进程明确地将该信号的配置设置为SIG_IGN，则调用进程的子进程将不产生僵死进程。这与其默认动作（SIG_DFL）“忽略”（见图10-1）不同。子进程在终止时，将其状态丢弃。如果调用进程随后调用一个wait函数，那么它将阻塞直到所有子进程都终止，然后该wait会返回−1，并将其errno设置为ECHILD。（此信号的默认配置是忽略，但这不会使上述语义起作用。必须将其配置明确指定为SIG_IGN才可以。）

-  如果将SIGCLD的配置设置为捕捉，则内核立即检查是否有子进程准备好被等待，如果有，则调用SIGCLD处理程序。

可靠信号术语和语义
~~~~~~~~~~~~~~~~~~

1. 当造成信号的事件发生时，为进程\ **产生**\ 一个信号（或向一个进程发送一个信号）。
2. 当一个信号产生时，内核通常在进程表中以某种形式设置一个标志。
3. 当对信号采取了这种动作时，我们说向进程\ **递送**\ 了一个信号。
4. 在信号产生（generation）和递送（delivery）之间的时间间隔内，称信号是\ **未决**\ 的（pending）。

kill和raise
~~~~~~~~~~~

kill函数将信号发送给进程或进程组，raise函数允许进程向自身发送信号。

.. code:: c

   #include <signal.h>

   int kill(pid_t pid, int signo);
   int raise(int signo);
   // 成功返回0，出错返回-1

调用\ ``raise(signo)``\ 等同于调用\ ``kill(getpid(), signo)``

kill的pid参数有4种不同的情况：

-  pid > 0：将信号发送给进程ID为pid的进程。

-  pid ==
   0：将信号发送给与发送进程属于同一进程组且发送进程具有发送权限的所有进程。

-  pid <
   0：将信号发送给进程组ID等于pid绝对值，而且发送进程发送权限的所有进程。

-  pid == −1：将信号发送给发送进程有权限向它们发送信号的所有进程。

..

   这里的术语“所有进程”不包括实现定义的系统进程集。对于大多数UNIX系统，系统进程集包括内核进程和init（pid为1）。

POSIX.1将信号编号0定义为空信号。如果signo参数是0，则kill仍执行正常的错误检查，但不发送信号。这常被用来确定一个特定进程是否仍然存在。如果向一个并不存在的进程发送空信号，则kill返回−1，errno被设置为ESRCH。

alarm和pause
~~~~~~~~~~~~

alarm函数用来设置一个定时器，在将来的某个时刻定时器会超时，并产生SIGALRM信号。如果忽略或者不捕捉该信号的话，默认动作是终止调用alarm函数的进程。

.. code:: c

   #include <unistd.h>

   unsigned int alarm(unsigned int seconds);
   // 返回0或者剩余的闹钟时间

参数seconds的值是产生信号SIGALRM需要经过的时钟秒数。当这一时刻到达时，信号由内核产生，由于进程调度的延迟，所以进程得到控制从而能够处理该信号还需要一个时间间隔。

调用alarm(0)可以取消上次的闹钟时间，并将其余留时间作为返回值。

pause函数使调用进程挂起直至捕捉到一个信号。

.. code:: c

   #include <unistd.h>

   int pause(void);
   // 返回-1，并将errno设置为EINTR

信号集
~~~~~~

POSIX.1定义数据类型sigset_t以包含一个信号集，并且定义了5个处理信号集的函数。

.. code:: c

   #include <signal.h>

   int sigemptyset(sigset_t* set);
   int sigfillset(sigset_t* set);
   int sigaddset(sigset_t* set, int signo);
   int sigdelset(sigset_t* set, int signo);
   // 成功返回0，出错返回-1

   int sigismember(const sigset_t* set, int signo);
   // 信号在信号集中返回1，否则返回0

-  sigemptyset初始化由set指向的信号集，清除其中所有信号。

-  sigfillset初始化由set指向的信号集，使其包括所有信号。

-  sigaddset将一个信号添加到已有的信号集中

-  sigdelset从信号集中删除一个信号。

sigprocmask
~~~~~~~~~~~

一个进程的信号屏蔽字规定了当前阻塞而不能递送给该进程的信号集。调用函数sigprocmask可以检测或更改，或同时进行检测和更改进程的信号屏蔽字。

.. code:: c

   #include <signal.h>

   int sigprocmask(int how, const sigset_t* restrict set, sigset_t* restrict oset);
   // 成功返回0，出错返回-1

-  若oset是非空指针，那么进程的当前信号屏蔽字通过oset返回。

-  若set是一个非空指针，则参数how指示如何修改当前信号屏蔽字。\ |image0|

-  如果set是个空指针，则不改变该进程的信号屏蔽字，how的值也无意义。

在调用sigprocmask后如果有任何未决的、不再阻塞的信号，则在sigprocmask返回前，至少将其中之一递送给该进程。

一个简单的示例：

.. code:: c

   #include "apue.h"
   #include <errno.h>

   void pr_mask(const char* str) { // print the signals
       sigset_t sigset;
       int errno_save;
       errno_save = errno;
       if (sigprocmask(0, NULL, &sigset) < 0) {
           err_ret("sigprocmask errno");
       } else {
           printf("%s", str);
           if (sigismember(&sigset, SIGINT)) {
               printf(" SIGINT");
           }
           if (sigismember(&sigset, SIGQUIT)) {
               printf(" SIGQUIT");
           }
           if (sigismember(&sigset, SIGUSR1)) {
               printf(" SIGUSR1");
           }
           if (sigismember(&sigset, SIGALRM)) {
               printf(" SIGALRM");
           }
           printf("\n");
       }
       errno = errno_save;
   }

sigpending
~~~~~~~~~~

sigpending函数返回一信号集。对于调用进程而言，其中的各信号是阻塞不能递送的，因而也一定是当前未决的。

.. code:: c

   #include <signal.h>

   int sigpending(sigset_t* set);
   // 成功返回0，出错返回-1

一个示例程序：

.. code:: c

   #include "apue.h"

   static void sig_quit(int);

   int main() {
       sigset_t newmask, oldmask, pendmask;
       if (signal(SIGQUIT, sig_quit) == SIG_ERR) {
           err_sys("can't catch SIGQUIT");
       }
       sigemptyset(&newmask);
       sigaddset(&newmask, SIGQUIT);
       if (sigprocmask(SIG_BLOCK, &newmask, &oldmask) < 0) {
           err_sys("SIG_BLOCK error");
       }
       sleep(5);
       if (sigpending(&pendmask) < 0) {
           err_sys("sigpending error");
       }
       if (sigismember(&pendmask, SIGQUIT)) {
           printf("\nSIGQUIT pending\n");
       }
       if (sigprocmask(SIG_SETMASK, &oldmask, NULL) < 0) {
           err_sys("SIG_SETMASK error");
       }
       printf("SIGQUIT unblocked\n");
       sleep(5);
       exit(0);
   }

   static void sig_quit(int signo) {
       printf("caught SIGQUIT\n");
       if (signal(SIGQUIT, SIG_DFL) == SIG_ERR) {
           err_sys("can't reset SIGQUIT");
       }
   }

sigaction
~~~~~~~~~

sigaction函数用来检查或修改与指定信号相关联的处理动作。

.. code:: c

   #include <signal.h>

   int sigaction(int signo, const struct sigaction* restrict act, struct sigaction* oact);
   // 成功返回0，出错返回-1

-  signo是要检测或修改其具体动作的信号编号。
-  若act指针非空，则要修改其动作。
-  如果oact指针非空，则通过oact指针返回该信号的上一个动作。

其中\ ``struct sigaction``\ 的结构如下：

.. code:: c

   struct sigaction {
       void (*sa_handler)(int);    // 信号处理程序地址或SIG_IGN或SIG_DEL
       sigset_t sa_mask;           // 额外的阻塞信号
       int sa_flags;               // signal options
       void (*sa_sigaction)(int, siginfo_t*, void*);   // 替代的信号处理程序
   };

如果sa_handler字段包含一个信号捕捉函数的地址（不是常量SIG_IGN或SIG_DFL），则sa_mask字段说明了一个信号集，在调用该信号捕捉函数之前，这一信号集要加到进程的信号屏蔽字中。仅当从信号捕捉函数返回时再将进程的信号屏蔽字恢复为原先值。

sa_flags字段指定对信号进行处理的各个选项。

sa_sigaction字段是一个替代的信号处理程序，在sigaction结构中使用了SA_SIGINFO标志时，使用该信号处理程序。对于sa_sigaction字段和sa_handler字段两者，实现可能使用同一存储区，所以应用只能一次使用这两个字段中的一个。

siginfo结构包含了信号产生原因的有关信息。该结构的大致样式如下所示。

.. code:: c

   struct siginfo {
       int si_signo;   /* signal number */
       int si_errno;   /* if nonzero, errno value from <errno.h> */
       int si_code;    /* additional info (depends on signal) */
       pid_t si_pid;   /* sending process ID */
       uid_t si_uid;   /* sending process real user ID */
       void* si_addr;  /* address that caused the fault */
       int si_status;          /* exit value or signal number */
       union sigval si_value;  /* application-specific value */
       /* possibly other fields also */
   };

sigsetjmp和siglongjmp
~~~~~~~~~~~~~~~~~~~~~

可以使用sigsetjmp和siglongjmp来在信号处理程序中进行非局部转移：

.. code:: c

   #include <setjmp.h>

   int sigsetjmp(segjmp_buf env, int savemask);
   // 直接调用返回0，从siglongjmp调用中返回非0
   void siglongjmp(sigjmp_buf env, int val);

-  调用sigsetjmp时，如果savemask非0，则sigsetjmp在env中保存进程的当前信号屏蔽字。

-  调用siglongjmp时，如果带非0savemask的sigsetjmp调用已经保存了env，则siglongjmp从其中恢复保存的信号屏蔽字。

sigsuspend
~~~~~~~~~~

使用sigsuspend可以在一个原子操作中先恢复信号屏蔽字，然后使进程休眠

.. code:: c

   #include <signal.h>

   int sigsuspend(const sigset_t* sigmask);
   // 返回-1，并将errno设置为EINTR

进程的信号屏蔽字设置为由sigmask指向的值。在捕捉到一个信号或发生了一个会终止该进程的信号之前，该进程被挂起。如果捕捉到一个信号而且从该信号处理程序返回，则sigsuspend返回，并且该进程的信号屏蔽字设置为调用sigsuspend之前的值。

一个示例程序：

.. code:: c

   #include "apue.h"

   volatile sig_atomic_t quitflag;

   static void sig_int(int signo) {
       if (signo == SIGINT) {
           printf("\ninterrupt\n");
       } else if (signo == SIGQUIT) {
           quitflag = 1;
       }
   }

   int main() {
       sigset_t newmask, oldmask, zeromask;
       if (signal(SIGINT, sig_int) == SIG_ERR) {
           err_sys("signal(SIGINT) error");
       }
       if (signal(SIGQUIT, sig_int) == SIG_ERR) {
           err_sys("signal(SIGQUIT) error");
       }
       sigemptyset(&zeromask);
       sigemptyset(&newmask);
       sigaddset(&newmask, SIGQUIT);
       if (sigprocmask(SIG_BLOCK, &newmask, &oldmask) < 0) {
           err_sys("SIG_BLOCK error");
       }
       while (quitflag == 0) {
           sigsuspend(&zeromask);
       }
       quitflag = 0;
       printf("hello, world!");
       if (sigprocmask(SIG_BLOCK, &oldmask, NULL) < 0) {
           err_sys("SIG_SETMASK error");
       }
       exit(0);
   }

abort
~~~~~

可以使用abort函数使程序异常终止：

.. code:: c

   #include <stdlib.h>

   void abort(void);

此函数将SIGABRT信号发送给调用进程（进程不应忽略此信号）。

ISO
C规定，调用abort将向主机环境递送一个未成功终止的通知，其方法是调用raise(SIGABRT)函数。ISO
C要求若捕捉到此信号而且相应信号处理程序返回，abort仍不会返回到其调用者。

system
~~~~~~

一个简单的示例程序：

.. code:: c

   #include "apue.h"

   static void sig_int(int signo) {
       printf("caught SIGINT\n");
   }

   static void sig_chld(int signo) {
       printf("caught SIGCHLD\n");
   }

   int main() {
       if (signal(SIGINT, sig_int) == SIG_ERR) {
           err_sys("signal(SIGINT) error");
       }
       if (signal(SIGCHLD, sig_chld) == SIG_ERR) {
           err_sys("signal(SIGCHLD) error");
       }
       if (system("/bin/ed") < 0) {
           err_sys("system() error");
       }
       exit(0);
   }

这样就能在ed编辑器的使用过程中捕捉特定的信号了(在此处是SIGINT和SIGCHLD)。

system函数一个进行了信号处理的实现：

.. code:: c

   #include <sys/wait.h>
   #include <errno.h>
   #include <signal.h>
   #include <unistd.h>

   int system(const char* cmdstring) {
       pid_t pid;
       int status;
       struct sigaction ignore, saveintr, savequit;
       sigset_t chldmask, savemask;
       if (cmdstring == NULL) {
           return 1;
       }
       ignore.sa_handler = SIG_IGN;    // ignore SIGINT and SIGQUIT
       sigemptyset(&ignore.sa_mask);
       ignore.sa_flags = 0;
       if (sigaction(SIGINT, &ignore, &saveintr) < 0) {
           return -1;
       }
       if (sigaction(SIGQUIT, &ignore, &savequit) < 0) {
           return -1;
       }
       sigemptyset(&chldmask);
       sigaddset(&chldmask, SIGCHLD);  // now block SIGCHLD
       if ((pid = fork()) < 0) {
           return -1;
       } else if (pid == 0) {
           sigaction(SIGINT, &saveintr, NULL);
           sigaction(SIGQUIT, &savequit, NULL);
           sigprocmask(SIG_SETMASK, &savemask, NULL);
           execl("/bin/sh", "sh", "-c", cmdstring, (char*)0);
           _exit(127);
       } else {
           while (waitpid(pid, &status, 0) < 0) {
               if (errno != EINTR) {
                   status = -1;
                   break;
               }
           }
       }
       if (sigaction(SIGINT, &saveintr, NULL) < 0) {
           return -1;
       }
       if (sigaction(SIGQUIT, &savequit, NULL) < 0) {
           return -1;
       }
       if (sigprocmask(SIG_SETMASK, &savemask, NULL) < 0) {
           return -1;
       }
       return status;
   }

sleep, nanosleep, clock_nanosleep
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c

   #include <unistd.h>

   unsigned int sleep(unsigned int seconds);
   // 返回0或者未休眠完的秒数

   #include <time.h>

   int nanosleep(const struct timespec* reqtp, struct timespec* remtp);
   // 若休眠到指定时间，返回0，出错返回-1

   int clock_nanoslepp(clockid_t clock_id, int flags, const struct timespec* reqtp, struct timespec* remtp);
   // 若休眠到指定时间，返回0，出错返回错误码

sleep函数使调用进程被挂起直到满足下面两个条件之一：

1. 已经过了seconds所指定的墙上时钟时间。

2. 调用进程捕捉到一个信号并从信号处理程序返回。

nanosleep函数与sleep函数类似，但提供了纳秒级的精度。

reqtp参数用秒和纳秒指定了需要休眠的时间长度。如果某个信号中断了休眠间隔，进程并没有终止，remtp参数指向的
timespec
结构就会被设置为未休眠完的时间长度。如果对未休眠完的时间并不感兴趣，可以把该参数置为NULL。

sigqueue
~~~~~~~~

使用排队信号必须做以下几个操作：

1. 使用sigaction函数安装信号处理程序时指定SA_SIGINFO标志。

2. 在sigaction结构的sa_sigaction成员中提供信号处理程序。

3. 使用sigqueue函数发送信号。

   .. code:: c

      #include <signal.h>

      int sigqueue(pid_t pid, int signo, const union sigval value);
      // 返回值：若成功，返回0；若出错，返回−1

   sigqueue函数只能把信号发送给单个进程，可以使用value参数向信号处理程序传递整数和指针值，除此之外，sigqueue函数与kill函数类似。

作业控制信号
~~~~~~~~~~~~

6个作业控制信号：

-  SIGCHLD 子进程已停止或终止。

-  SIGCONT 如果进程已停止，则使其继续运行。

-  SIGSTOP 停止信号（不能被捕捉或忽略）。

-  SIGTSTP 交互式停止信号。

-  SIGTTIN 后台进程组成员读控制终端。

-  SIGTTOU 后台进程组成员写控制终端。

当对一个进程产生 4
种停止信号（SIGTSTP、SIGSTOP、SIGTTIN或SIGTTOU）中的任意一种时，对该进程的任一未决SIGCONT信号就被丢弃。

与此类似，当对一个进程产生SIGCONT信号时，对同一进程的任一未决停止信号被丢弃。

如果进程是停止的，则SIGCONT的默认动作是继续该进程；否则忽略此信号。

   当对一个停止的进程产生一个SIGCONT信号时，该进程继续，即使该信号被阻塞或忽略。

信号名与编号
~~~~~~~~~~~~

使用psignal函数可移植地打印与信号编号对应的字符串。

.. code:: c

   #include <signal.h>

   void psignal(int signo, const char *msg);

字符串msg（通常是程序名）输出到标准错误文件，后面跟随一个冒号和一个空格，再后面对该信号的说明，最后是一个换行符。如果msg为NULL，只有信号说明部分输出到标准错误文件。

如果sigaction信号处理程序中有siginfo结构，可以使用psiginfo函数打印信号信息：

.. code:: c

   #include <signal.h>

   void psiginfo(const siginfo_t *info, const char *msg);

可以使用strsignal函数获取信号的字符描述部分：

.. code:: c

   #include <string.h>

   char *strsignal(int signo);
   // 返回值：指向描述该信号的字符串的指针

Solaris提供一对函数，一个函数将信号编号映射为信号名，另一个则反之。

.. code:: c

   #include <signal.h>

   int sig2str(int signo, char *str);
   int str2sig(const char *str, int *signop);
   // 成功返回0，出错返回−1

.. |image0| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00218.jpg
