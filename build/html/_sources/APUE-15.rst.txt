进程间通信
----------

进程间通信，InterProcess Communication, 简称IPC。

管道
~~~~

可以通过pipe函数来创建一个管道：

.. code:: c

   #include <unistd.h>

   int pipe(int fd[2]);
   // 成功返回0，出错返回-1

成功返回后，fd[0]为读而打开，fd[1]为写而打开。fd[1]的输出是fd[0]的输入。

fstat函数对管道的每一端都返回一个FIFO类型的文件描述符。可以用S_ISFIFO宏来测试管道。

通常，进程会先调用pipe，接着调用fork，从而创建从父进程与子进程之间的IPC通道。

-  对于父进程到子进程的通道。父进程关闭管道的读端(fd[0])，子进程关闭写端(fd[1])。
-  对于从子进程到父进程的管道，父进程关闭写端(fd[1])，子进程关闭读端(fd[0])。

当读（read）一个写端已被关闭的管道时，在所有数据都被读取后，read返回0，表示文件结束。

当写（write）一个读端已被关闭的管道时，产生信号SIGPIPE。如果忽略该信号或者捕捉该信号并从其处理程序返回，则write返回−1，errno设置为EPIPE。

一个简单的示例：

.. code:: c

   #include "apue.h"

   int main() {
       int n;
       int fd[2];
       pid_t pid;
       char line[MAXLINE];
       if (pipe(fd) < 0) {
           err_sys("pipe error");
       }
       if ((pid = fork()) < 0) {
           err_sys("fork error");
       } else if (pid > 0) {
           close(fd[0]);
           write(fd[1], "hello world!\n", 12);
       } else {
           close(fd[1]);
           n = read(fd[0], line, MAXLINE);
           write(STDOUT_FILENO, line, n);
       }
       exit(0);
   }

TELL_WAIT系列函数的使用管道的实现：

.. code:: c

   #include "apue.h"

   static int pfd1[2], pfd2[2];

   void TELL_WAIT(void) {
       if (pipe(pfd1) < 0 || pipe(pfd2) < 0) {
           err_sys("pipe error");
       }
   }

   void TELL_PARENT(pid_t pid) {
       if (write(pfd2[1], "c", 1) != 1) {
           err_sys("write error");
       }
   }

   void WAIT_PARENT(void) {
       char c;
       if (read(pfd1[0], &c, 1) != 1) {
           err_sys("read error");
       }
       if (c != 'p') {
           err_quit("WAIT_PARENT: incorrect data");
       }
   }

   void TELL_CHILD(pid_t pid) {
       if (write(pfd1[1], "p", 1) != 1) {
           err_sys("write error");
       }
   }

   void WAIT_CHILD(void) {
       char c;
       if (read(pfd2[0], &c, 1) != 1) {
           err_sys("read error");
       }
       if (c != 'c') {
           err_quit("WAIT_CHILD: incorrect data");
       }
   }

popen和pclose
~~~~~~~~~~~~~

.. code:: c

   #include <stdio.h>

   FILE* popen(const char* cmdstring, const char* type);
   // 成功返回文件指针，出错返回NULL
   int pclose(FILE* fp);
   // 成功返返回cmdstring的终止状态，出错返回-1

函数popen先执行fork，然后调用exec执行cmdstring，并且返回一个标准I/O文件指针。

-  如果type是“r”，则文件指针连接到cmdstring的标准输出。

-  如果type是“w”，则文件指针连接到cmdstring的标准输入，

pclose函数关闭标准I/O流，等待命令终止，然后返回shell的终止状态。如果shell不能被执行，则pclose返回的终止状态与shell执行exit(127)一样。

示例程序(popen和pclose的实现)：

.. code:: c

   #include "apue.h"
   #include <errno.h>
   #include <fcntl.h>
   #include <sys/wait.h>

   static pid_t* childpid = NULL;  // pointer to array allocated at run-time
   static int maxfd;

   FILE* popen(const char* cmdstring, const char* type) {
       int i;
       int pfd[2];
       pid_t pid;
       FILE* fp;
       if ((type[0] != 'r' && type[0] != 'w') || type[1] != 0) {
           errno = EINVAL;
           return NULL;
       }
       if (childpid == NULL) {
           maxfd = open_max();
           if ((childpid = calloc(maxfd, sizeof(pid_t))) == NULL) {
               return NULL;
           }
       }
       if (pipe(pfd) < 0) {
           return NULL;
       }
       if (pfd[0] >= maxfd || pfd[1] >= maxfd) {
           close(pfd[0]);
           close(pfd[1]);
           errno = EMFILE;
           return NULL;
       }
       if ((pid = fork()) < 0) {
           return NULL;
       } else if (pid == 0) {
           if (*type == 'r') {
               close(pfd[0]);
               if (pfd[1] != STDOUT_FILENO) {
                   dup2(pfd[1], STDOUT_FILENO);
                   close(pfd[1]);
               }
           } else {
               close(pfd[1]);
               if (pfd[0] != STDIN_FILENO) {
                   dup2(pfd[0], STDIN_FILENO);
                   close(pfd[0]);
               }
           }
           /* close all descriptors in childpid[] */
           /* to comply with POSIX.1 */
           for (i = 0; i < maxfd; i++) {
               if (childpid[i] > 0) {
                   close(i);
               }
           }
           execl("/bin/sh", "sh", "-c", cmdstring, (char*)0);
           _exit(127);
       }
       /* parent */
       if (*type == 'r') {
           close(pfd[1]);
           if ((fp = fdopen(pfd[0], type)) == NULL) {
               return NULL;
           }
       } else {
           close(pfd[0]);
           if ((fp = fdopen(pfd[1], type)) == NULL) {
               return NULL;
           }
       }
       childpid[fileno(fp)] = pid; // remember child pid for this fd
       return fp;
   }

   int pclose(FILE* fp) {
       int fd, stat;
       pid_t pid;
       if (childpid == NULL) {
           errno = EINVAL;
           return -1;
       }
       fd = fileno(fp);
       if (fd > maxfd) {
           errno = EINVAL;
           return -1;
       }
       if ((pid = childpid[fd]) == 0) {
           errno = EINVAL;
           return -1;
       }
       childpid[fd] = 0;
       if (fclose(fp) == EOF) {
           return -1;
       }
       while (waitpid(pid, &stat, 0) < 0) {
           if (errno != EINTR) {
               return -1;
           }
       }
       return stat;
   }

协同进程
~~~~~~~~

UNIX系统过滤程序从标准输入读取数据，向标准输出写数据。当一个过滤程序既产生某个过滤程序的输入，又读取该过滤程序的输出时，它就变成了\ **协同进程**\ （coprocess）。

FIFO
~~~~

FIFO又被称为\ **命名管道**\ 。

未命名的管道只能在两个相关的进程之间使用，而且这两个相关的进程还要有一个共同的创建了它们的祖先进程。但是，通过FIFO，不相关的进程也能交换数据。

可以使用mkfifo函数来创建一个FIFO：

.. code:: c

   #include <sys/stat.h>

   int mkfifo(const char* path, mode_t mode);
   int mkfifoat(int fd, const char* path, mode_t mode);
   // 成功返回0，出错返回-1

mode参数的可选值与open函数中mode参数的相同.

带at类型的函数依旧是原来的模式：

-  如果path参数指定的是绝对路径名，则fd参数会被忽略，mkfifoat函数的行为和mkfifo类似。
-  如果path参数指定的是相对路径名，则fd参数是一个打开目录的有效文件描述符，。
-  如果path参数指定的是相对路径名，并且fd参数有一个特殊值AT_FDCWD，则路径名以当前目录开始，mkfifoat和mkfifo类似。

创建FIFO后，要使用open来打开该FIFO。

open FIFO时的非阻塞标志(O_NONBLOCK)的影响：

-  在一般情况下（没有指定O_NONBLOCK），只读open要阻塞到某个其他进程为写而打开这个FIFO为止。只写open要阻塞到某个其他进程为读而打开它为止。

-  如果指定了O_NONBLOCK，则只读open立即返回。如果没有进程为读而打开一个FIFO，那么只写open将返回−1，并将errno设置成ENXIO。

XSI IPC
~~~~~~~

有三种XSI IPC：消息队列、信号量和共享存储器。

标识符和键
^^^^^^^^^^

每个内核中的IPC结构（消息队列、信号量或共享存储段）都用一个非负整数的标识符（identifier）加以引用。

每个IPC对象都与一个键（key，对应的类型为key_t）相关联，将这个键作为该对象的外部名。

使客户进程和服务器进程在同一IPC结构上汇聚的方法：

1. 服务器进程可以指定键IPC_PRIVATE创建一个新IPC结构，将返回的标识符存放在某处（如一个文件）以便客户进程取用。键IPC_PRIVATE保证服务器进程创建一个新IPC结构。
2. 在一个公用头文件中定义一个客户进程和服务器进程都认可的键。然后服务器进程指定此键创建一个新的IPC结构。
3. 客户进程和服务器进程认同一个路径名和项目ID（项目ID是0～255之间的字符值），接着，调用函数ftok将这两个值变换为一个键。然后在方法（2）中使用此键。

.. code:: c

   #include <sys/ipc.h>

   key_t ftok(const char* path, int fd);
   // 成功返回键，出错返回(key_t)-1

决不能指定IPC_PRIVATE作为键来引用一个现有队列，这个特殊的键值总是用于创建一个新队列。

如果希望创建一个新的IPC结构，而且要确保没有引用具有同一标识符的一个现有IPC结构，那么必须在flag中同时指定IPC_CREAT和IPC_EXCL位。这样做了以后，如果IPC结构已经存在就会造成出错，返回EEXIST。

权限结构
~~~~~~~~

XSI
IPC为每个IPC结构都关联了ipc_perm结构。该结构规定了权限和所有者，它至少包括下列成员：

.. code:: c

   struct ipc_perm {
       uid_t uid;      /* owner's effective user id */
       gid_t gid;      /* owner's effective group id */
       uid_t cuid;     /* creator's effective user id */
       gid_t cgid;     /* creator's effective group id */
       mode_t mode;    /* access modes */
       /* more fi*/
   };

mode字段是S_IRUSR，S_IWUSR等9个值的组合。

消息队列
~~~~~~~~

消息队列是消息的链接表，存储在内核中，由消息队列标识符(队列ID)标识。

msgget用于创建一个新队列或打开一个现有队列。msgsnd将新消息添加到队列尾端。msgrcv用于从队列中取消息。

每个队列都有一个msqid_ds结构与其相关联：

.. code:: c

   struct msqid_ds {
       struct ipc_perm msg_perm;   // 权限结构，可参考上一部分
       msgqnum_t msg_qnum;         // num of messages on queue
       msglen_t msg_qbytes;        // max num of bytes on queue
       pid_t msg_lspid;            // pid of last msgsnd()
       pid_t msg_lrpid;            // pid of last msgrcv()
       time_t msg_stime;           // last-msgsnd() time
       time_t msg_rtime;           // last-msgrcv() time
       time_t msg_ctime;           // last change time
   };

msgget的原型如下：

.. code:: c

   #include <sys/msg.h>

   int msgget(key_t key, int flag);
   // 成功返回消息队列ID，出错返回-1

创建新队列时，会对msqid_ds结构进行初始化：

-  初始化ipc-perm结构。其中的mode成员会按flag中的相应权限位设置。flag的值如下：

   .. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512091501715.png
      :alt: image-20210512091501715

      image-20210512091501715

-  msg_qnum、msg_lspid、msg_lrpid、msg_stime和msg_rtime都设置为0。

-  msg_ctime设置为当前时间。

-  msg_qbytes设置为系统限制值。

返回的队列ID可以用于其他几个函数。

msgctl是队列相关的垃圾桶函数(即能干很多事情)：

.. code:: c

   #include <sys/msg.h>

   int msgctl(int msqid, int cmd, struct msqid_ds* buf);
   // 成功返回0，出错返回-1

msqid即为队列ID。

cmd的取值有以下几种情况：

-  IPC_STAT：将此队列的msqid_ds结构存放在buf指向的结构中。

-  IPC_SET：将字段 msg_perm.uid、msg_perm.gid、msg_perm.mode 和
   msg_qbytes从buf指向的结构复制到与这个队列相关的msqid_ds结构中。

-  IPC_RMID：从系统中删除该消息队列以及仍在该队列中的所有数据。这种删除立即生效。仍在使用这一消息队列的其他进程在它们下一次试图对此队列进行操作时，将得到EIDRM错误。

..

   后面两条命令有权限要求：要么进程的有效用户ID等于msg_perm.cuid或msg_perm.uid，要么具有超级用户特权。并且只有超级用户才能增加msg_qbytes的值。

msgsnd用于将数据放到消息队列中：

.. code:: c

   #include <sys/msg.h>

   int msgsnd(int msqid, const void* ptr, size_t nbytes, int flag);
   // 成功返回0，出错返回-1

msqid为队列ID。

ptr指向一个长整型数，长整型数后面的是消息数据。长整型数表示消息类型(为正)，后面消息数据的长度由nbytes表明。如发送的最长消息为512字节，则可定义以下结构：

.. code:: c

   struct mymesg {
       long mtype;
       char mtext[512];
   };

ptr就是一个指向mymesg结构的指针。

接收者可以使用消息类型(mtype字段)以非先进先出的次序取消息。

参数flag的值可以指定为IPC_NOWAIT。若消息队列已满（或者是队列中的消息总数等于系统限制值，或队列中的字节总数等于系统限制值），msgsnd立即出错返回EAGAIN。

如果没有指定IPC_NOWAIT，则进程会一直阻塞到：有空间可以容纳要发送的消息；或者从系统中删除了此队列(返回EIDRM错误)；或者捕捉到一个信号，并从信号处理程序返回(返回EINTR错误)。

对删除消息队列的处理不是很完善。每个消息队列没有维护引用计数器（打开文件有这种计数器），所以在队列被删除以后，仍在使用这一队列的进程在下次对队列进行操作时会出错返回。

msgsnd返回成功时，消息队列相关的msqid_ds结构会随之更新，表明调用的进程ID（msg_lspid）、调用的时间（msg_stime）以及队列中新增的消息（msg_qnum）。

msgrcv用于从队列中取消息：

.. code:: c

   #include <sys/msg.h>

   ssize_t msgrcv(int msqid, const void* ptr, size_t nbytes, long type, int flag);
   // 成功返回消息数据的长度，出错返回-1

ptr参数指向一个长整型数（其中存储的是返回的消息类型），其后跟随的是存储实际消息数据的缓冲区。nbytes指定数据缓冲区的长度。

若返回的消息长度大于nbytes，而且在flag中设置了MSG_NOERROR位，则该消息会被截断。如果没有设置这一标志，而消息又太长，则出错返回E2BIG（消息仍留在队列中）。

参数type可以指定想要哪一种消息：

-  type == 0：返回队列中的第一个消息。
-  type > 0：返回队列中消息类型为type的第一个消息。
-  type <
   0：返回队列中消息类型值小于等于type绝对值的消息，如果这种消息有若干个，则取类型值最小的消息。

flag值可以指定为IPC_NOWAIT，使操作不阻塞。如果没有所指定类型的消息可用，则msgrcv返回−1，error设置为ENOMSG。

如果没有指定IPC_NOWAIT，则进程会一直阻塞到有了指定类型的消息可用，或者从系统中删除了此队列（返回−1，error设置为EIDRM），或者捕捉到一个信号并从信号处理程序返回（这会导致msgrcv返回−1，errno设置为EINTR）。

msgrcv成功执行时，内核会更新与该消息队列相关联的msgid_ds结构，以指示调用者的进程ID（msg_lrpid）和调用时间（msg_rtime），并指示队列中的消息数减少了1个（msg_qnum）。

信号量
~~~~~~

XSI信号量并非是单个非负值，而是含有一个或多个信号量的集合。创建信号量时，要制定集合中信号量的个数。

内核为每个信号量集合维护一个semid_ds结构：

.. code:: c

   struct semid_ds {
       struct ipc_perm sem_perm;   // 权限结构
       unsigned short sem_nsems;   // num of semaphores in set
       time_t sem_otime;           // last-semop() time
       time_t sem_ctime;           // last-change time
   };

每个信号量由一个无名结构表示，至少包含以下成员：

.. code:: c

   struct {
       unsigned short semval;  // semaphore value, always >= 0
       pid_t sempid;           // pid for last operation
       unsigned short semncnt; // num of processes awaiting semval > curval
       unsigned short semzcnt; // num of processes awaiting semval == 0
   };

可以通过semget函数来获得一个信号量ID：

.. code:: c

   #include <sys/sem.h>

   int semget(key_t key, int nsems, int flag);
   // 成功返回信号量ID，出错返回-1

创建一个新集合时，内核会这样初始化semid_ds结构：

-  icp_perm结构的初始化与消息队列类似。
-  sem_otime设置为0。
-  sem_ctime设置为当前时间。
-  sem_nsems设置为nsems参数。

信号量同样有一个垃圾桶函数semctl：

.. code:: c

   #include <sys/sem.h>

   int semctl(int semid, int semnum, int cmd, .../* union semun arg */);
   // 返回值见下方

union semun的结构如下：

.. code:: c

   union semun {
       int val;                // for SETVAL
       struct semid_ids* buf;  // for IPC_STAT and ICP_SET
       unsigned short* array;  // for GETALL and SETALL
   };

cmd参数的可能取值：

-  IPC_STAT：对此集合取semid_ds结构，并存储在由arg.buf指向的结构中。
-  IPC_SET：按arg.buf指向的结构中的值，设置与此集合相关的结构中的sem_perm.uid、sem_perm.gid和sem_perm.mode字段。
-  IPC_RMID：从系统中删除该信号量集合。这种删除是立即发生的。删除时仍在使用此信号量集合的其他进程，在它们下次试图对此信号量集合进行操作时，将出错返回EIDRM。
-  GETVAL：返回成员semnum的semval值。
-  SETVAL：设置成员semnum的semval值。该值由arg.val指定。
-  GETPID：返回成员semnum的sempid值。
-  GETNCNT：返回成员semnum的semncnt值。
-  GETZCNT：返回成员semnum的semzcnt值。
-  GETALL：取该集合中所有的信号量值。这些值存储在arg.array指向的数组中。
-  SETALL：将该集合中所有的信号量值设置成arg.array指向的数组中的值。

..

   ICP_SET和ICP_RMID的权限要求和消息队列中的心痛

对于除GETALL以外的所有GET命令，semctl函数都返回相应值。对于其他命令，若成功则返回值为0，若出错，则设置errno并返回−1。

函数semop自动执行信号量集合上的操作数组：

.. code:: c

   #include <sys/sem.h>

   int semop(int semid, struct sembuf semoparray[], size_t nops);
   // 返回值：若成功，返回0；若出错，返回−1

其中sembuf结构如下：

.. code:: c

   struct sembuf {
       unsigned short sem_num; // 信号量的在信号量集合中序号，[0, nsems-1]
       short sem_op;           // operation(neg, 0, or pos)
       short sem_flg;          // ICP_NOWAIT, SEM_UNDO
   };

1. 若sem_op为正值。信号量的值加sem_op。如果指定了undo标志，则变为减去sem_op。

2. 若sem_op为负值。若信号量值>=sem_op的绝对值，则信号量的值加sem_op(加一个负值就相当于减小)。如果指定了undo标志，则变为减去sem_op。若信号量值<sem_op的绝对值，则：

   -  若指定了IPC_NOWAIT，则semop出错返回EAGAIN。

   -  若未指定IPC_NOWAIT，则信号量的semncnt值加1，然后调用进程被挂起直至下列事件之一发生：

      -  此信号量值变成大于等于sem_op的绝对值。然后信号量的semncnt值减1，并且从信号量值中减去sem_op的绝对值。如果指定了undo标志，变为加。
      -  从系统中删除了此信号量。在这种情况下，函数出错返回EIDRM。
      -  进程捕捉到一个信号，并从信号处理程序返回，在这种情况下，此信号量的semncnt值减1，并且函数出错返回EINTR。

3. 若sem_op为0，这表示调用进程希望等待到该信号量值变成0。

   -  如果信号量值当前是0，则此函数立即返回。
   -  如果信号量值非0，则：

      -  若指定了IPC_NOWAIT，则出错返回EAGAIN。
      -  若未指定IPC_NOWAIT，则该信号量的semzcnt加1，然后调用进程被挂起，直至下列的一个事件发生：

         -  此信号量值变成0。此信号量的semzcnt值减1。
         -  从系统中删除了此信号量。在这种情况下，函数出错返回EIDRM。
         -  进程捕捉到一个信号，并从信号处理程序返回。在这种情况下，此信号量的semzcnt值减1，并且函数出错返回EINTR。

semop函数具有原子性，它或者执行数组中的所有操作，或者一个也不做。

共享存储
~~~~~~~~

XSI共享存储和内存映射的文件的不同之处在于，共享存储没有相关的文件。XSI共享存储段是内存的匿名段。

内核为每个共享内存段维护着一个shmid_ds结构：

.. code:: c

   struct shmid_ids {
       struct ipc_perm shm_perm;   /* 权限结构 */
       size_t shm_segsz;           /* size of segment in bytes */
       pid_t shm_lpid;             /* pid of last shmop() */
       pid_t  shm_cpid;            /* pid of creator */
       shmatt_t shm_nattch;        /* number of current attaches */
       time_t shm_atime;           /* last-attach time */
       time_t shm_dtime;           /* last-detach time */
       time_t shm_ctime;           /* last-change time */
   };

可以通过shmget函数来获取一个共享存储标识符。

.. code:: c

   #include <sys/shm.h>

   int shmget(key_t key, size_t size, int flag);
   // 成功返回共享存储ID，出错返回-1

创建一个新共享存储段时，内核会这样初始化shmid_ds结构：

-  icp_perm结构的初始化与消息队列类似。
-  shm_lpid、shm_nattach、shm_atime和shm_dtime都设置为0。
-  sem_ctime设置为当前时间。
-  shm_segsz设置为size参数。

参数size是该共享存储段的长度，以字节为单位。实现通常将其向上取为系统页长的整倍数。但是，若应用指定的size值并非系统页长的整倍数，那么最后一页的余下部分是不可使用的。

如果正在创建一个新段（通常在服务器进程中），则必须指定其size。如果正在引用一个现存的段（一个客户进程），则将size指定为0。当创建一个新段时，段内的内容初始化为0。

共享存储同样有一个垃圾桶函数shmctl：

.. code:: c

   #include <sys/shm.h>

   int shmctl(int shmid, int cmd, struct shmid_ds* buf);
   // 成功返回0，出错返回-1

cmd参数的可能值：

-  IPC_STAT：取此段的shmid_ds结构，并将它存储在由buf指向的结构中。

-  IPC_SET：按buf指向的结构中的值设置与此共享存储段相关的shmid_ds
   结构中的下列3个字段：shm_perm.uid、shm_perm.gid和shm_perm.mode。

-  IPC_RMID：从系统中删除该共享存储段。因为每个共享存储段维护着一个连接计数（shmid_ds结构中的shm_nattch字段），所以除非使用该段的最后一个进程终止或与该段分离，否则不会实际上删除该存储段。不管此段是否仍在使用，该段标识符都会被立即删除，所以不能再用shmat与该段连接。

-  SHM_LOCK：在内存中对共享存储段加锁。此命令只能由超级用户执行。

-  SHM_UNLOCK：解锁共享存储段。此命令只能由超级用户执行。

..

   IPC_SET和ICP_RMID的权限要求与之前相同。

   最后两个命令并非SUS的组成部分。Linux和Solaris提供了这两个命令。

一旦创建了一个共享存储段，进程就可调用shmat将其连接到它的地址空间中。

.. code:: c

   #include <sys/shm.h>

   void* shmat(int shmid, const void* addr, int flag);
   // 成功返回指向共享存储段的指针，出错返回-1

共享存储段连接到调用进程的哪个地址上与addr参数以及flag中是否指定SHM_RND位有关：

-  如果addr为0，则此段连接到由内核选择的第一个可用地址上。这是推荐的使用方式。
-  如果addr非0，并且没有指定SHM_RND，则此段连接到addr所指定的地址上。
-  如果addr非0，并且指定了SHM_RND，则此段连接到（addr−(addr mod
   SHMLBA)）所表示的地址上。SHM_RND命令的意思是“取整”。SHMLBA的意思是“低边界地址倍数”，它总是2的乘方。该算式是将地址向下取最近1个SHMLBA的倍数。

如果在flag中指定了SHM_RDONLY位，则以只读方式连接此段，否则以读写方式连接此段。

当对共享存储段的操作已经结束时，则调用shmdt与该段分离。注意，这并不从系统中删除其标识符以及其相关的数据结构。该标识符仍然存在，直至某个进程（一般是服务器进程）带IPC_RMID命令的调用shmctl特地删除它为止。

.. code:: c

   #include <sys/shm.h>

   int shmdt(const void* addr);
   // 成功返回0，出错返回-1

如果成功，shmdt将使相关shmid_ds结构中的shm_nattch计数器值减1。

POSIX信号量
~~~~~~~~~~~

POSIX信号量有两种形式：命名的和未命名的。它们的差异在于创建和销毁的形式上，但其他工作一样。

-  未命名信号量只存在于内存中，并要求能使用信号量的进程必须可以访问内存。这意味着它们只能应用在同一进程中的线程，或者不同进程中已经映射相同内存内容到它们的地址空间中的线程。

-  命名信号量可以通过名字访问，可以被任何已知它们名字的进程中的线程使用。

我们可以调用sem_open函数来创建一个新的命名信号量或者使用一个现有信号量。

.. code:: c

   #include <semaphore.h>

   sem_t* sem_open(const char* name, int oflag, ... /* mode_t mode, unsigned int value */ );
   // 成功返回指向信号量的指针，出错返回SEM_FAILED

当使用一个现有的命名信号量时，仅需指定信号量的名字，并将oflag设置为0。

当oflag参数有O_CREAT标志集时，如果命名信号量不存在，则创建一个新的。如果它已经存在，则会被使用，但是不会有额外的初始化发生。

当我们指定O_CREAT标志时，需要提供两个额外的参数。mode参数指定谁可以访问信号量。mode的取值和打开文件的权限位相同：用户读、用户写、用户执行、组读、组写、组执行、其他读、其他写和其他执行。赋值给信号量的权限可以被调用者的文件创建屏蔽字修改。

在创建信号量时，value参数用来指定信号量的初始值。它的取值是0～SEM_VALUE_MAX。

如果我们想确保创建的是信号量，可以设置oflag参数为O_CREAT \|
O_EXCL。如果信号量已经存在，会导致sem_open失败。

信号量命名规范：

-  名字的第一个字符应该为斜杠（/）。
-  名字不应包含其他斜杠以此避免实现定义的行为。
-  信号量名字的最大长度是实现定义的。名字不应该长于_POSIX_NAME_MAX。

可以调用sem_close函数来释放信号量相关的资源：

.. code:: c

   #include <semaphore.h>

   int sem_close(sem_t* sem);
   // 成功返回0，出错返回-1

如果进程没有首先调用sem_close而退出，那么内核将自动关闭任何打开的信号量。

可以使用sem_unlink函数来销毁一个命名信号量。

.. code:: c

   #include <semaphore.h>

   int sem_unlink(const char *name);
   // 返回值：若成功，返回0；若出错，返回-1

sem_unlink函数删除信号量的名字。如果没有打开的信号量引用，则该信号量会被销毁。否则，销毁将延迟到最后一个打开的引用关闭。

可以使用sem_wait或者sem_trywait函数来实现信号量的减1操作(即P操作)。

.. code:: c

   #include <semaphore.h>

   int sem_trywait(sem_t* sem);
   int sem_wait(sem_t* sem);
   // 成功返回0，出错返回−1

使用sem_wait函数时，如果信号量计数是0就会阻塞。直到成功使信号量减1或者被信号中断时才返回。

调用sem_trywait时，如果信号量是0，则不会阻塞，而是会返回−1并且将errno置为EAGAIN。

还可以使用sem_timedwait函数来指定最大等待时间：

.. code:: c

   #include <semaphore.h>
   #include <time.h>

   int sem_timedwait(sem_t* restrict sem, const struct timespec* restrict tsptr);
   // 返回值：若成功，返回0；若出错，返回−1

如果超时到期并且信号量计数没能减1，sem_timedwait将返回-1且将errno设置为ETIMEDOUT。

可以调用sem_post函数使信号量值增1(即V操作)：

.. code:: c

   #include <semaphore.h>

   int sem_post(sem_t* sem);
   // 成功返回0，出错返回−1

调用sem_post时，如果在调用sem_wait（或者sem_timedwait）中发生进程阻塞，那么进程会被唤醒并且被sem_post增1的信号量计数会再次被sem_wait（或者sem_timedwait）减1。

可以调用sem_init函数来创建一个未命名的信号量。

.. code:: c

   #include <semaphore.h>

   int sem_init(sem_t* sem, int pshared, unsigned int value);
   // 成功返回0，出错返回−1

pshared参数表明是否在多个进程中使用信号量。如果是，需要设置成一个非0值。

value参数指定了信号量的初始值。

可以调用sem_destroy函数销毁未命名的信号量：

.. code:: c

   #include <semaphore.h>

   int sem_destroy(sem_t* sem);
   // 成功返回0，出错返回−1

可以使用sem_getvalue函数来检索信号量值：

.. code:: c

   #include <semaphore.h>

   int sem_getvalue(sem_t *restrict sem, int *restrict valp);
   // 成功返回0，出错返回−1

调用成功后，valp指向的整数值将包含信号量值。

   但是我们试图要使用我们刚读出来的值的时候，信号量的值可能已经变了。除非使用额外的同步机制来避免这种竞争，否则sem_getvalue函数只能用于调试。

用信号量来实现自己的锁结构：

.. code:: c

   // slock.h
   #include <semaphore.h>
   #include <fcntl.h>
   #include <limits.h>
   #include <sys/stat.h>

   struct slock {
       sem_t* semp;
       char name[_POSIX_NAME_MAX];
   };

   struct slock* s_alloc();
   void s_free(struct slock*);
   int s_lock(struct slock*);
   int s_trylock(struct slock*);
   int s_unlock(struct slock*);

.. code:: c

   #include "slock.h"
   #include <stdlib.h>
   #include <stdio.h>
   #include <unistd.h>
   #include <errno.h>

   struct slock* s_alloc() {
       struct slock* sp;
       static int cnt;
       if ((sp = malloc(sizeof(struct slock))) == NULL) {
           return NULL;
       }
       do {
           snprintf(sp->name, sizeof(sp->name), "/%ld.%d", (long)getpid(), cnt++);
           sp->semp = sem_open(sp->name, O_CREAT | O_EXCL, S_IRWXU, 1);
       } while ((sp->semp == SEM_FAILED) && (errno == EEXIST));
       if (sp->semp == SEM_FAILED) {
           free(sp);
           return NULL;
       }
       sem_unlink(sp->name);
       return sp;
   }

   void s_free(struct slock* sp) {
       sem_close(sp->semp);
       free(sp);
   }

   int s_lock(struct slock* sp) {
       return sem_wait(sp->semp);
   }

   int s_trylock(struct slock* sp) {
       return sem_trywait(sp->semp);
   }

   int s_unlock(struct slock* sp) {
       return sem_post(sp->semp);
   }
