线程控制
--------

线程属性
~~~~~~~~

可以通过pthread_attr_t类型来控制pthread_create函数新建线程的属性：

pthread_attr_t类型的初始化与反初始化：

.. code:: c

   #include <pthread.h>

   int pthread_attr_init(pthread_attr_t* attr);
   int pthread_attr_destroy(pthread_attr_t* attr);
   // 成功返回0，出错返回错误编号

还可以通过以下函数来获取和设置pthread_attr_t结构中的detachstate线程属性：

.. code:: c

   #include <pthread.h>

   int pthread_attr_getdetachstate(const pthread_attr_t* restrict attr, int* detachstate);
   int pthread_attr_setdetachstate(pthread_attr_t* attr, int* detachstate);
   // 成功返回0，出错返回错误编号

detachstate有两个合法值：PTHREAD_CREATE_DETACHED(以分离状态启动线程)和PTHREAD_CREATE_JOINABLE(正常启动程序)。

一个示例：

.. code:: c

   #include "apue.h"
   #include <pthread.h>

   int makethread(void* (*fn)(void*), void* arg) {
       int err;
       pthread_t tid;
       pthread_attr_t attr;
       err = pthread_attr_init(&attr);
       if (err != 0) {
           return err;
       }
       err = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
       if (err == 0) {
           err = pthread_create(&tid, &attr, fn, arg);
       }
       pthread_attr_destroy(&attr);
       return err;
   }

可以通过以下函数对线程栈属性进行管理：

.. code:: c

   #include <pthread.h>

   int pthread_attr_getstack(const pthread_attr_t* restrict attr, void** restrict stackaddr, size_t* restrict stacksize);
   int pthread_attr_setstack(pthread_attr_t* attr, void* stackaddr, size_t stacksize);
   // 成功返回0，出错返回错误编号

如果线程栈的虚地址空间都用完了，那可以使用malloc或者mmap来为可替代的栈分配空间，并用pthread_attr_setstack函数来改变新建线程的栈位置。

可以通过以下函数来读取或设置线程属性stacksize：

.. code:: c

   #include <pthread.h>

   int pthread_attr_getstacksize(const pthread_attr_t* restrict attr, size_t* restrict stacksize);
   int pthread_attr_setstacksize(pthread_attr_t* attr, size_t stacksize);
   // 成功返回0，出错返回错误编号

线程属性guardsize控制着线程栈末尾之后用以避免栈溢出的扩展内存的大小。这个属性默认值是由具体实现来定义的，但常用值是系统页大小。可以把guardsize线程属性设置为0，不允许属性的这种特征行为发生：在这种情况下，不会提供警戒缓冲区。同样，如果修改了线程属性stackaddr，系统就认为我们将自己管理栈，进而使栈警戒缓冲区机制无效，这等同于把guardsize线程属性设置为0。

.. code:: c

   #include <pthread.h>

   iint pthread_attr_getguardsize(const pthread_attr_t* restrict attr, size_t* restrict guardsize);
   int pthread_attr_setguardsize(pthread_attr_t* attr, size_t guardsize);
   // 成功返回0，出错返回错误编号

如果guardsize线程属性被修改了，操作系统可能会把它取为页大小的整数倍。如果线程的栈指针溢出到警戒区域，应用程序就可能通过信号接收到出错信息。

同步属性
~~~~~~~~

互斥量属性
^^^^^^^^^^

使用pthread_mutexaddr_t类型进行控制互斥量属性：

.. code:: c

   #include <pthread.h>

   int pthread_mutexattr_init(pthread_mutexattr_t* attr);
   int pthread_mutexattr_destroy(pthread_mutexattr_t* attr);
   // 成功返回0，出错返回错误编号

互斥量的进程共享属性：

.. code:: c

   #include <pthread.h>

   int pthread_mutexattr_getpshared(const pthread_mutexattr_t* restrict attr, int* restrict pshared);
   int pthread_mutexattr_setpshared(pthread_mutexattr_t* attr, int pshared);
   // 成功返回0，出错返回错误编号

默认情况下，进程共享互斥量属性设置为PTHREAD_PROCESS_PRIVATE，在当前进程中的多个线程可以访问同一个同步对象。

如果进程共享互斥量属性设置为PTHREAD_PROCESS_SHARED，从多个进程彼此之间共享的内存数据块中分配的互斥量就可以用于这些进程的同步。

互斥量的健壮属性：

.. code:: c

   #include <pthread.h>

   int pthread_mutexattr_getrobust(const pthread_mutexattr_t* restrict attr, int* restrict robust);
   int pthread_mutexattr_setrobust(pthread_mutexattr_t* attr, int robust);
   // 成功返回0，出错返回错误编号

健壮属性取值有两种可能的情况：

-  默认为PTHREAD_MUTEX_STALLED，这意味着持有互斥量的进程终止时不需要采取特别的动作。
-  另一个取值是PTHREAD_MUTEX_ROBUST。这个值将导致线程调用pthread_mutex_lock获取锁，而该锁被另一个进程持有，但它终止时并没有对该锁进行解锁，此时线程会阻塞，pthread_mutex_lock返回EOWNERDEAD而不是0。应用程序可以通过这个特殊的返回值获知，若有可能，不管它们保护的互斥量状态如何，都需要进行恢复。

如果应用状态无法恢复，在线程对互斥量解锁以后，该互斥量将处于永久不可用状态。为了避免这样的问题，线程可以调用pthread_mutex_consistent函数，指明与该互斥量相关的状态在互斥量解锁之前是一致的。

.. code:: c

   #include <pthread.h>

   int pthread_mutex_consistent(pthread_mutex_t *mutex);
   // 成功返回0，出错返回错误编号

如果线程没有先调用pthread_mutex_consistent就对互斥量进行了解锁，那么其他试图获取该互斥量的阻塞线程就会得到错误码ENOTRECOVERABLE。如果发生这种情况，互斥量将不再可用。线程通过提前调用pthread_mutex_consistent，能让互斥量正常工作，这样它就可以持续被使用。

互斥量的类型属性：

.. code:: c

   #include <pthread.h>

   int pthread_mutexattr_gettype(const pthread_mutexattr_t* restrict attr,int* restrict type);
   int pthread_mutexattr_settype(pthread_mutexattr_t* attr, int type);
   // 成功返回0，出错返回错误编号

type参数的可能取值：

-  PTHREAD_MUTEX_NORMAL：标准互斥量类型，不做任何特殊的错误检查或死锁检测。

-  PTHREAD_MUTEX_ERRORCHECK：提供错误检查。

-  PTHREAD_MUTEX_RECURSIVE
   此互斥量类型允许同一线程在互斥量解锁之前对该互斥量进行多次加锁。递归互斥量维护锁的计数，在解锁次数和加锁次数不相同的情况下，不会释放锁。所以，如果对一个递归互斥量加锁两次，然后解锁一次，那么这个互斥量将依然处于加锁状态，对它再次解锁以前不能释放该锁。

-  PTHREAD_MUTEX_DEFAULT：提供默认特性和行为。操作系统在实现它的时候可以把这种类型自由地映射到其他互斥量类型中的一种。例如，Linux
   3.2.0把这种类型映射为普通的互斥量类型，而FreeBSD
   8.0则把它映射为错误检查互斥量类型。

读写锁属性
^^^^^^^^^^

用pthread_rwlockattr_t类型控制读写锁的属性。

属性的初始化和反初始化：

.. code:: c

   #include <pthread.h>

   int pthread_rwlockattr_init(pthread_rwlockattr_t* attr);
   int pthread_rwlockattr_destroy(pthread_rwlockattr_t* attr);
   // 成功返回0，出错返回错误编号

读写锁支持的唯一属性是进程共享属性：

.. code:: c

   #include <pthread.h>

   int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t* restrict attr, int* restrict pshared);
   int pthread_rwlockattr_setpshared(pthread_rwlockattr_t* attr, int pshared);
   // 成功返回0，出错返回错误编号

条件变量属性
^^^^^^^^^^^^

通过pthread_condattr_t类型控制条件变量的属性：

属性的初始化和反初始化：

.. code:: c

   #include <pthread.h>

   int pthread_condattr_init(pthread_condattr_t* attr);
   int pthread_condattr_destroy(pthread_condattr_t* attr);
   // 成功返回0，出错返回错误编号

SUS定义了条件变量的两个属性：进程共享和时钟属性：

.. code:: c

   #include <pthread.h>

   int pthread_condattr_getpshared(const pthread_condattr_t* restrict attr, int* restrict pshared);
   int pthread_condattr_setpshared(pthread_condattr_t* attr, int pshared);
   int pthread_condattr_getclock(const pthread_condattr_t* restrict attr, clockid_t* restrict clock_id);
   int pthread_condattr_setclock(pthread_condattr_t* attr, clockid_t* restrict clock_id);
   // 成功返回0，出错返回错误编号

时钟属性控制计算pthread_cond_timedwait函数的超时参数（tsptr）时采用的是哪个时钟。

屏障属性
^^^^^^^^

通过pthread_barrierattr_t类型控制条件变量的属性：

属性的初始化和反初始化：

.. code:: c

   #include <pthread.h>

   int pthread_barrierattr_init(pthread_barrierattr_t* attr);
   int pthread_barrierattr_destroy(pthread_barrierattr_t* attr);
   // 成功返回0，出错返回错误编号

目前定义的屏障属性只有进程共享属性：

.. code:: c

   #include <pthread.h>

   int pthread_barrierattr_getpshared(const pthread_barrierattr_t* restrict attr, int* restrict pshared);
   int pthread_barrierattr_setpshared(pthread_barrierattr_t* attr, int pshared);
   // 成功返回0，出错返回错误编号

重入
~~~~

如果一个函数对多个线程来说是可重入的，就说这个函数就是\ **线程安全**\ 的。但这并不能说明对信号处理程序来说该函数也是可重入的。如果函数对异步信号处理程序的重入是安全的，那么就可以说函数是\ **异步信号安全**\ 的。

POSIX.1提供了以线程安全的方式管理FILE对象的方法。可以使用flockfile和ftrylockfile获取给定FILE对象关联的锁。这个锁是递归的：当你占有这把锁的时候，还是可以再次获取该锁，而且不会导致死锁。

   所有操作FILE对象的标准I/O例程的动作行为必须看起来就像它们内部调用了flockfile和funlockfile。

.. code:: c

   #include <stdio.h>

   int ftrylockfile(FILE* fp);
   // 成功返回0，若不能获取锁则返回非0数值
   void flockfile(FILE* fp);
   void funlockfile(FILE* fp);

为了避免锁带来的读写单个字符的性能下降，出现了不加锁版本的基于字符的标准I/O例程：

.. code:: c

   #include <stdio.h>

   int getchar_unlocked(void);
   int getc_unlocked(FILE* fp);
   // 成功返回下一个字符，遇到文件尾或者出错返回EOF
   int putchar_unlocked(int c);
   int putc_unlocked(int c, FILE* fp);
   // 成功返回c，出错返回EOF

可重入(线程安全)版本的getenv函数实现：

.. code:: c

   #include <string.h>
   #include <errno.h>
   #include <pthread.h>
   #include <stdlib.h>

   extern char** environ;
   pthread_mutex_t env_mutex;
   static pthread_once_t init_done = PTHREAD_ONCE_INIT;

   static void thread_init() {
       pthread_mutexattr_t attr;
       pthread_mutexattr_init(&attr);
       pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
       pthread_mutex_init(&env_mutex, &attr);
       pthread_mutexattr_destroy(&attr);
   }

   int getenv_r(const char* name, char* buf, int buflen) {
       int i, len, olen;
       pthread_once(&init_done, thread_init);
       len = strlen(name);
       pthread_mutex_lock(&env_mutex);
       for (i = 0; environ[i] != NULL; i++) {
           if ((strncmp(name, environ[i], len) == 0) && (environ[i][len] == '=')) {
               olen = strlen(&environ[i][len+1]);
               if (olen >= buflen) {
                   pthread_mutex_unlock(&env_mutex);
                   return ENOSPC;
               }
               strcpy(buf, &environ[i][len+1]);
               pthread_mutex_unlock(&env_mutex);
               return 0;
           }
       }
       pthread_mutex_unlock(&env_mutex);
       return ENOENT;
   }

线程特定数据
~~~~~~~~~~~~

线程特定数据（thread-specific data），也称为线程私有数据（thread-private
data），是存储和查询某个特定线程相关数据的一种机制。每个线程可以访问它自己单独的数据副本，而不需要担心与其他线程的同步访问问题。

在分配线程特定数据之前，需要创建与该数据关联的键。这个键将用于获取对线程特定数据的访问。使用pthread_key_create创建一个键。

.. code:: c

   #include <pthread.h>

   int pthread_key_create(pthread_key_t* keyp, void (*destructor)(void*));
   // 成功返回0，出错返回错误编号

除了创建键以外，pthread_key_create
可以为该键关联一个可选择的析构函数。当这个线程退出时，如果数据地址已经被置为非空值，那么析构函数就会被调用，它唯一的参数就是该数据地址。

当线程调用pthread_exit或者线程执行返回，正常退出时，析构函数就会被调用。同样，线程取消时，只有在最后的清理处理程序返回之后，析构函数才会被调用。如果线程调用了exit、_exit、_Exit或abort，或者出现其他非正常的退出时，就不会调用析构函数。

可以使用pthread_key_delete取消键和线程特定数据值之间的关联关系：

.. code:: c

   #include <pthread.h>

   int pthread_key_delete(pthread_key_t key);
   // 成功返回0，出错返回错误编号

通过使用pthread_once来保证只初始化一次：

.. code:: c

   #include <pthread.h>

   pthread_once_t initflag = PTHREAD_ONCE_INIT;

   int pthread_once(pthread_once_t* initflag, void (*initfn)(void));
   // 成功返回0，出错返回错误编号

initflag必须是一个非本地变量（如全局变量或静态变量），而且必须初始化为
PTHREAD_ONCE_INIT。

键创建之后，可以使用pthread_setspecific函数将键和特定数据类型关联起来：

.. code:: c

   #include <pthread.h>

   void* pthread_getspecific(pthread_key_t key);
   // 返回线程特定数据值，若没有值与该键关联则返回NULL
   int pthread_setspecific(pthread_key_t key, const void* value);
   // 成功返回0，出错返回错误编号

线程安全的getenv的兼容版本：

.. code:: c

   #include <limits.h>
   #include <string.h>
   #include <pthread.h>
   #include <stdlib.h>

   #define MAXSTRINGSZ 4096
   static pthread_key_t key;
   static pthread_once_t init_done = PTHREAD_ONCE_INIT;
   pthread_mutex_t env_mutex = PTHREAD_MUTEX_INITIALIZER;
   extern char** environ;

   static void thread_init() {
       pthread_key_create(&key, free);
   }

   char* getenv(const char* name) {
       int i, len;
       char* envbuf;
       pthread_once(&init_done, thread_init);
       pthread_mutex_lock(&env_mutex);
       envbuf = (char*) pthread_getspecific(key);
       if (envbuf == NULL) {
           envbuf = malloc(MAXSTRINGSZ);
           if (envbuf == NULL) {
               pthread_mutex_unlock(&env_mutex);
               return NULL;
           }
           pthread_setspecific(key, envbuf);
       }
       len = strlen(name);
       for (i = 0; environ[i] != NULL; i++) {
           if ((strncmp(name, environ[i], len) == 0) && (environ[i][len] == '=')) {
               strncpy(envbuf, &environ[i][len+1], MAXSTRINGSZ - 1);
               pthread_mutex_unlock(&env_mutex);
               return envbuf;
           }
       }
       pthread_mutex_unlock(&env_mutex);
       return NULL;
   }

取消选项
~~~~~~~~

有两个线程属性并没有包含在pthread_attr_t结构中，它们是可取消状态和可取消类型。这两个属性影响着线程在响应pthread_cancel函数调用时所呈现的行为。

可取消状态属性可以是PTHREAD_CANCEL_ENABLE，也可以是PTHREAD_CANCEL_DISABLE。线程可以通过调用pthread_setcancelstate修改它的可取消状态。

.. code:: c

   #include <pthread.h>

   int pthread_setcancelstate(int state, int* oldstate);
   // 返回值：若成功，返回0；否则，返回错误编号

在默认情况下，线程在取消请求发出以后还是继续运行，直到线程到达某个取消点。取消点是线程检查它是否被取消的一个位置，如果取消了，则按照请求行事。

一些会导致请求点出现的函数请参考APUE第三版P362-363。

线程启动时默认的可取消状态是
PTHREAD_CANCEL_ENABLE。当状态设为PTHREAD_CANCEL_DISABLE时，对pthread_cancel的调用并不会杀死线程。相反，取消请求对这个线程来说还处于挂起状态，当取消状态再次变为PTHREAD_CANCEL_ENABLE时，线程将在下一个取消点上对所有挂起的取消请求进行处理。

可以通过pthread_testcancle函数来添加自己的取消点：

.. code:: c

   #include <pthread.h>

   void pthread_testcancle(void);

调用pthread_testcancel时，如果有某个取消请求正处于挂起状态，而且取消并没有置为无效，那么线程就会被取消。但是，如果取消被置为无效，pthread_testcancel调用就没有任何效果了。

默认的取消类型为推迟取消(PTHREADCANCEL_DEFERRED)。调用pthread_cancel以后，在线程到达取消点之前，并不会出现真正的取消。

还可以将取消类型设置为异步取消(PTHREAD_CANCEL_ASYNCHRONOUS)。使用异步取消时，线程可以在任意时间撤消，不是非得遇到取消点才能被取消。

可以通过调用pthread_setcanceltype来修改取消类型。

.. code:: c

   #include <pthread.h>

   int pthread_setcanceltype(int type, int* oldtype);
   // 成功返回0，出错返回错误编号

线程和信号
~~~~~~~~~~

每个线程都有自己的信号屏蔽字，但是信号的处理是进程中所有线程共享的。

sigprocmask的行为在多线程的进程中并没有定义，线程必须使用pthread_sigmask来设置信号屏蔽字：

.. code:: c

   #include <signal.h>

   int pthread_sigmask(int how, const sigset_t* restrict set, sigset_t* restrict oset);
   // 成功返回0，出错返回错误编号

线程可以通过调用sigwait等待一个或多个信号的出现（个人感觉和sigsuspend很像）：

.. code:: c

   #include <signal.h>

   int sigwait(const sigset_t *restrict set, int *restrict signop);
   // 成功返回0，出错返回错误编号

要把信号发送给进程，可以调用kill。要把信号发送给线程，可以调用pthread_kill。

.. code:: c

   #include <signal.h>

   int pthread_kill(pthread_t thread, int signo);
   // 成功返回0，出错返回错误编号

一个示例程序：

.. code:: c

   #include "apue.h"
   #include <pthread.h>

   int quitflag;
   sigset_t mask;

   pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
   pthread_cond_t waitloc = PTHREAD_COND_INITIALIZER;

   void* thr_fn(void* arg) {
       int err, signo;
       for ( ; ; ) {
           err = sigwait(&mask, &signo);
           if (err != 0) {
               err_exit(err, "sigwait failed");
           }
           switch (signo) {
               case SIGINT:
                   printf("\ninterrupt\n");
                   break;
               case SIGQUIT:
                   pthread_mutex_lock(&lock);
                   quitflag = 1;
                   pthread_mutex_unlock(&lock);
                   pthread_cond_signal(&waitloc);
                   return 0;
               default:
                   printf("unexpected signal %d\n", signo);
                   exit(1);
           }
       }
   }

   int main() {
       int err;
       sigset_t oldmask;
       pthread_t tid;
       sigemptyset(&mask);
       sigaddset(&mask, SIGINT);
       sigaddset(&mask, SIGQUIT);
       if ((err = pthread_sigmask(SIG_BLOCK, &mask, &oldmask)) != 0) {
           err_exit(err, "SIG_BLOCK failed");
       }
       err = pthread_create(&tid, NULL, thr_fn, 0);
       if (err != 0) {
           err_exit(err, "pthread_create failed");
       }
       pthread_mutex_lock(&lock);
       while (quitflag == 0) {
           pthread_cond_wait(&waitloc, &lock);
       }
       pthread_mutex_unlock(&lock);
       quitflag = 0;
       if (sigprocmask(SIG_SETMASK, &oldmask, NULL) < 0) {
           err_sys("SIG_SETMASK error");
       }
       exit(0);
   }

线程与fork
~~~~~~~~~~

当线程调用fork时，就为子进程创建了整个进程地址空间的副本。

子进程通过继承整个地址空间的副本，还从父进程那儿继承了每个互斥量、读写锁和条件变量的状态。如果父进程包含一个以上的线程，子进程在fork返回以后，如果紧接着不是马上调用exec的话，就需要清理锁状态。

可以调用pthread_atfork函数安装最多3个帮助清理锁的函数。

.. code:: c

   #include <pthread.h>

   int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));
   // 成功返回0，出错返回错误编号

prepare处理程序由父进程在fork创建子进程前调用。这个处理程序的任务是获取父进程定义的所有锁。

parent处理程序是在fork创建子进程以后、返回之前在父进程上下文中调用的。这个处理程序的任务是对prepare处理程序获取的所有锁进行解锁。

child处理程序在fork返回之前在子进程上下文中调用。child处理程序也必须释放prepare处理程序获取的所有锁。

parent和child处理程序是以它们注册时的顺序进行调用的，而prepare处理程序的调用顺序与它们注册时的顺序相反。这样可以允许多个模块注册它们自己的fork处理程序，而且可以保持锁的层次。

线程与I/O
^^^^^^^^^

可参考第三章笔记的\ **原子操作**\ 部分的pread和pwrite函数。
