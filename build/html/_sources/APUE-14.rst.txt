高级I/O
-------

非阻塞I/O
~~~~~~~~~

非阻塞I/O使我们可以发出open、read和write这样的I/O操作，但不会永远阻塞。如果这种操作不能完成，则调用立即出错返回，表示该操作如继续执行将阻塞。

对于一个给定的描述符，有两种为其指定非阻塞I/O的方法：

1. 如果调用open获得描述符，则可指定O_NONBLOCK标志。

2. 对于已经打开的一个描述符，则可调用fcntl函数打开O_NONBLOCK文件状态标志。

一个简单的示例：

.. code:: c

   #include "apue.h"
   #include <errno.h>
   #include <fcntl.h>

   char buf[500000];

   int main() {
       int ntowrite, nwrite;
       char* ptr;
       ntowrite = read(STDIN_FILENO, buf, sizeof(buf));
       fprintf(stderr, "read %d bytes\n", ntowrite);
       set_fl(STDOUT_FILENO, O_NONBLOCK);
       ptr = buf;
       while (ntowrite > 0) {
           errno = 0;
           nwrite = write(STDOUT_FILENO, ptr, ntowrite);
           fprintf(stderr, "nwrite = %d, errno = %d\n", nwrite, errno);
           if (nwrite > 0) {
               ptr += nwrite;
               ntowrite -= nwrite;
           }
       }
       clr_fl(STDOUT_FILENO, O_NONBLOCK);
       exit(0);
   }

记录锁
~~~~~~

记录锁（record
locking）的功能是：当一个进程正在读或修改文件的某个部分时，使用记录锁可以阻止其他进程修改同一文件区。

fcntl记录锁
^^^^^^^^^^^

.. code:: c

   #include <fcntl.h>

   int fcntl(int fd, int cmd, .../* struct flock* flockptr */);
   // 成功的返回值依赖于cmd，出错返回-1

对于记录锁，cmd是F_GETLK、F_SETLK或F_SETLKW，第三个参数使用flockptr。

flockptr是一个指向flock结构的指针。

.. code:: c

   struct flock {
       short l_type;   /* 锁类型: F_RDLCK, F_WRLCK, or F_UNLCK */
       short l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END */
       off_t l_start;  /* 相对l_whence的偏移量 */
       off_t l_len;    /* 加锁区域长度, 0代表一直到EOF */
       pid_t l_pid;    /* cmd为F_GETLK，返回时, pid代表的进程持有的锁能阻塞当前进程 */
   };

共享读锁（l_type为L_RDLCK）和独占性写锁（L_WRLCK）的基本规则是：任意多个进程在一个给定的字节上可以有一把共享的读锁，但是在一个给定字节上只能有一个进程有一把独占写锁。

cmd三种取值的作用：

-  F_GETLK：判断由flockptr所描述的锁是否会被另外一把锁所阻塞。如果存在一把锁，它阻止创建由flockptr所描述的锁，则该现有锁的信息将重写flockptr指向的信息。如果不存在这种情况，则除了将l_type设置为F_UNLCK之外，flockptr所指向结构中的其他信息保持不变。

-  F_SETLK：设置由flockptr所描述的锁。如果我们试图获得一把读锁（l_type
   为F_RDLCK）或写锁（l_type为F_WRLCK），而兼容性规则阻止系统给我们这把锁，那么fcntl会立即出错返回，此时errno设置为EACCES或EAGAIN。此命令也用来清除由flockptr指定的锁（l_type为F_UNLCK）。

-  F_SETLKW：这个命令是F_SETLK的阻塞版本(W for
   Wait)。如果所请求的读锁或写锁因另一个进程当前已经对所请求区域的某部分进行了加锁而不能被授予，那么调用进程会被置为休眠。如果请求创建的锁已经可用，或者休眠由信号中断，则该进程被唤醒。

在设置或释放文件上的一把锁时，系统按要求组合或分裂相邻区。

锁的隐含继承和释放
^^^^^^^^^^^^^^^^^^

记录锁的自动继承和释放的3条规则：

1. 锁与进程和文件两者相关联。当一个进程终止时，它所建立的锁全部释放；无论一个描述符何时关闭，该进程通过这一描述符引用的文件上的任何一把锁都会释放。
2. fork产生的子进程不继承父进程所设置的锁。
3. 在执行exec后，新程序可以继承原执行程序的锁。但是如果对一个文件描述符设置了\ **执行时关闭标志**\ ，那么当作为exec的一部分关闭该文件描述符时，将释放相应文件的所有锁。

建议性锁和强制性锁
^^^^^^^^^^^^^^^^^^

建议性锁不能阻止对文件有写权限的任何其他进程写这个文件。

强制性锁会让内核检查每一个open、read和write，验证调用进程是否违背了正在访问的文件上的某一把锁。

   强制性锁有时也称为强迫方式锁（enforcement-mode locking）。

对一个特定文件打开其设置组ID位、关闭其组执行位便开启了对该文件的强制性锁机制。

强制性锁对其他进程的read和write的影响：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210511135401997.png
   :alt: image-20210511135401997

   image-20210511135401997

I/O多路转接
~~~~~~~~~~~

select和pselect
^^^^^^^^^^^^^^^

select函数使我们可以执行I/O多路转接。传给select的参数告诉内核：

-  我们所关心的描述符；
-  对于每个描述符我们所关心的条件（是否想从一个给定的描述符读，是否想写一个给定的描述符，是否关心一个给定描述符的异常条件）；
-  愿意等待多长时间（可以永远等待、等待一个固定的时间或者根本不等待）。

从select返回时，内核告诉我们：

-  已准备好的描述符的总数量；
-  对于读、写或异常这3个条件中的每一个，哪些描述符已准备好。

.. code:: c

   #include <sys/select.h>

   int select(int maxfdp1, fd_set* restrict readfds, fd_set* restrict writefds, fd_set* restrict exceptfds, struct timeval* restrict tvptr);
   // 返回准备就绪的描述符数目，超时返回0，出错返回−1

tvptr参数指定愿意等待的时间长度：

-  tvptr ==
   NULL：永远等待。如果捕捉到一个信号则中断此无限期等待。当所指定的描述符中的一个已准备好或捕捉到一个信号则返回。如果捕捉到一个信号，则select返回-1，errno设置为EINTR。

-  tvptr->tv_sec == 0 && tvptr->tv_usec ==
   0：不等待。测试所有指定的描述符并立即返回。可以用来查询多个描述符状态的状态。

-  tvptr->tv_sec != 0 \|\| tvptr->tv_usec !=
   0：当指定的描述符之一已准备好，或当指定的时间值已经超过时立即返回。如果在超时到期时还没有一个描述符准备好，则返回0。这种等待同样可被捕捉到的信号中断。

readfds、writefds和exceptfds都是指向描述符集的指针。

每个描述符集存储在一个fd_set数据类型中。它可以为每一个可能的描述符保持一位。

fd_set数据类型的相关操作：

.. code:: c

   #include <sys/select.h>

   int FD_ISSET(int fd, fd_set* fdset);
   // 若fd在描述符集中返回非0值，否则返回0
   void FD_CLR(int fd, fd_set* fdset);
   void FD_SET(int fd, fd_set* fdset);
   void FD_ZERO(fd_set* fdset);

这些接口可实现为宏或函数。

-  调用FD_ZERO将一个fd_set变量的所有位设置为0。

-  要开启描述符集中的一位，可以调用FD_SET。

-  调用FD_CLR可以清除一位。

-  可以调用FD_ISSET测试描述符集中的一个指定位是否已打开。

maxfdp1参数是三个描述符集中最大文件描述符的编号值加1。这个参数用于提高遍历文件描述符的效率。

select的返回值：

-  返回值-1：出错了。如在所指定的描述符一个都没准备好时捕捉到一个信号。在此种情况下，一个描述符集都不修改。

-  返回值0：没有描述符准备好就超时了。此时，所有描述符集都会置0。

-  正的返回值：说明了已经准备好的描述符数。该值是3个描述符集中已准备好的描述符数之和。

..

   如果在一个描述符上碰到了文件尾端，则select会认为该描述符是可读的。然后调用read，它返回0，这是UNIX系统指示到达文件尾端的方法。

POSIX.1定义了一个select的变体，称为pselect：

.. code:: c

   #include <sys/select.h>

   int pselect(int maxfdp1, fd_set* restrict readfds, fd_set* restrict writefds, fd_set* restrict exceptfds, const struct timespec* restrict tsptr, const sigset_t* restrict sigmask);
   // 返回准备就绪的描述符数目，超时返回0，出错返回−1

-  select的超时值用timeval结构指定，而pselect使用timespec结构(更加精确)。

-  pselect的超时值被声明为const，这保证了调用pselect不会改变此值。

-  sigmask指定信号屏蔽字，在调用pselect时，以原子操作的方式安装该信号屏蔽字。在返回时，恢复以前的信号屏蔽字。（为NULL时和select相同）

poll
~~~~

.. code:: c

   #include <poll.h>

   int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
   // 返回准备就绪的描述符数目，超时返回0，出错返回-1

fdarray数组中的每个元素指定一个描述符编号以及我们对该描述符感兴趣的条件。nfds参数指定数组的长度。

其中struct pollfd结构如下：

.. code:: c

   struct pollfd {
       int fd;         /* file descriptor to check, or < 0 to ignore */
       short events;   /* events of interest on fd */
       short revents;  /* events that occurred on fd */
   };

events成员的取值：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210511142942948.png
   :alt: image-20210511142942948

   image-20210511142942948

timeout的可能取值：

-  timeout ==
   -1：永远等待。当所指定的描述符中的一个已准备好，或捕捉到一个信号时返回。如果捕捉到一个信号，则poll返回-1，errno设置为EINTR。

-  timeout == 0：不等待。测试所有描述符并立即返回。

-  timeout >
   0：等待timeout毫秒。当指定的描述符之一已准备好，或timeout到期时立即返回。如果timeout到期时还没有一个描述符准备好，则返回值是0。

异步I/O
~~~~~~~

POSIX异步I/O接口为对不同类型的文件进行异步I/O提供了一套一致的方法。

异步I/O基于aiocb(AIO控制块)结构，该结构\ **至少**\ 包括下面这些字段：

.. code:: c

   struct aiocb {
       int aio_fildes;                 /* file descriptor */
       off_t aio_offset;               /* file offset for I/O */
       volatile void* aio_buf;         /* buffer for I/O */
       size_t aio_nbytes;              /* number of bytes to transfer */
       int aio_reqprio;                /* priority */
       struct sigevent aio_sigevent;   /* signal information */
       int aio_lio_opcode;             /* operation for list I/O */
   };

aio_sigevent字段控制在I/O事件完成后，如何通知应用程序。

这个字段通过sigevent结构来描述：

.. code:: c

   struct sigevent {
       int sigev_notify;                           /* notify type */
       int sigev_signo;                            /* signal number */
       union sigval sigev_value;                   /* notify argument */
       void (*sigev_notify_function)(union sigval);/* notify function */
       pthread_attr_t* sigev_notify_attributes;    /* notify attrs */
   };

sigev_notify字段控制通知的类型：

-  SIGEV_NONE：异步I/O请求完成后，不通知进程。

-  SIGEV_SIGNAL：异步I/O请求完成后，产生由sigev_signo字段指定的信号。

-  SIGEV_THREAD：当异步I/O请求完成时，由sigev_notify_function字段指定的函数被调用。sigev_value字段被传入作为它的唯一参数。除非sigev_notify_attributes字段被设定为pthread属性结构的地址，且该结构指定了一个另外的线程属性，否则该函数将在分离状态下的一个单独的线程中执行。

调用aio_read函数来进行异步读操作，调用aio_write函数来进行异步写操作。

.. code:: c

   #include <aio.h>

   int aio_read(struct aiocb* aiocb);
   int aio_write(struct aiocb* aiocb);
   // 成功返回0，出错返回−1

可以调用aio_fsync函数强制所有等待中的异步操作不等待而写入持久化的存储中。

.. code:: c

   #include <aio.h>

   int aio_fsync(int op, struct aiocb* aiocb);
   // 成功返回0，出错返回−1

AIO控制块中的aio_fildes字段指定了其异步写操作被同步的文件。

-  如果op参数设定为O_DSYNC，那么操作执行起来就会像调用了fdatasync一样。
-  如果op参数设定为O_SYNC，那么操作执行起来就会像调用了fsync一样。

可以调用aio_error函数获取异步读、写或者同步操作的完成状态：

.. code:: c

   #include <aio.h>

   int aio_error(const struct aiocb* aiocb);

返回值有4种情况：

-  0：异步操作成功完成。需要调用aio_return函数获取操作返回值。

-  −1：对aio_error的调用失败。

-  EINPROGRESS：异步读、写或同步操作仍在等待。

-  其他的返回值是相关的异步操作失败返回的错误码。

如果异步操作成功，可以调用aio_return函数来获取异步操作的返回值。

.. code:: c

   #include <aio.h>

   ssize_t aio_return(const struct aiocb* aiocb);

返回值：

-  如果aio_return函数本身失败，会返回−1，并设置errno。
-  其他情况下，它将返回异步操作的结果，即会返回read、write或者fsync在被成功调用时可能返回的结果。

需要在异步操作成功后再调用aio_return函数。操作完成之前的结果是未定义的。

对每个异步操作只能调用一次aio_return。调用后操作系统就可以释放掉包含了I/O操作返回值的记录。

如果完成了所有事务后还有异步操作未完成时，可以调用aio_suspend函数来阻塞进程，直到操作完成。

.. code:: c

   #include <aio.h>

   int aio_suspend(const struct aiocb* const list[], int nent, const struct timespec* timeout);
   // 成功返回0，出错返回−1

返回值：

-  如果被一个信号中断，返回-1，并将errno设置为EINTR。
-  如果在没有任何I/O操作完成的情况下，阻塞的时间超过了函数中可选的timeout参数所指定的时间限制，那么aio_suspend将返回-1，并将errno设置为EAGAIN。
-  如果有任何I/O操作完成，aio_suspend将返回0。

如果在我们调用aio_suspend操作时，所有的异步I/O操作都已完成，那么aio_suspend将在不阻塞的情况下直接返回。

timeout参数为空指针时表示不设置任何时间限制。

list参数是一个指向AIO控制块数组的指针，nent参数表明了数组中的条目数。数组中的空指针会被跳过，其他条目都必须指向已用于初始化异步I/O操作的AIO控制块。

可以使用aio_cancel函数尝试取消不想再完成的等待中的异步I/O操作。

.. code:: c

   #include <aio.h>

   int aio_cancel(int fd, struct aiocb* aiocb);

返回值：

-  AIO_ALLDONE：所有操作在尝试取消它们之前已经完成。

-  AIO_CANCELED：所有要求的操作已被取消。

-  AIO_NOTCANCELED：至少有一个要求的操作没有被取消。

-  -1：对aio_cancel的调用失败，错误码将被存储在errno中。

fd参数指定了未完成的异步I/O操作的文件描述符。如果aiocb参数为NULL，系统将会尝试取消所有该文件上未完成的异步I/O操作。

   无法保证系统能够取消正在进程中的任何操作。

如果异步I/O操作被成功取消，对相应的AIO控制块调用aio_error函数将会返回错误ECANCELED。

如果操作不能被取消，那么相应的AIO控制块不会因为对aio_cancel的调用而被修改。

lio_listio函数既可以以同步方式调用，也可以以异步方式调用。该函数提交一系列由一个AIO控制块列表描述的I/O请求。

.. code:: c

   #include <aio.h>

   int lio_listio(int mode, struct aiocb* restrict const list[restrict], int nent, struct sigevent* restrict sigev);
   // 成功返回0，出错返回−1

mode参数的可能值：

-  LIO_WAIT：函数将在所有由列表指定的I/O操作完成后返回。在这种情况下，sigev参数将被忽略。

-  LIO_NOWAIT：函数将在I/O请求入队后立即返回。进程将在所有I/O操作完成后，按照sigev参数指定的，被异步地通知。如果不想被通知，可以把sigev设定为NULL。

      每个AIO控制块本身也可能启用了在各自操作完成时的异步通知。被sigev参数指定的异步通知是在此之外另加的，并且只会在所有的I/O操作完成后发送。

list参数指向AIO控制块列表，该列表指定了要运行的I/O操作。nent参数指定了数组中的元素个数。列表中的NULL条目将被忽略。

struct
aiocb结构中的aio_lio_opcode字段指定了该操作是一个读操作（LIO_READ）、写操作（LIO_WRITE），还是将被忽略的空操作（LIO_NOP）。读操作会按照对应的AIO控制块被传给了aio_read函数来处理。写操作会按照对应的AIO控制块被传给了aio_write函数来处理。

一个异步I/O的示例程序：

.. code:: c

   #include "apue.h"
   #include <ctype.h>
   #include <fcntl.h>
   #include <aio.h>
   #include <errno.h>

   #define BSZ 4096
   #define NBUF 8

   enum rwop {
       UNUSED = 0,
       READ_PENDING = 1,
       WRITE_PENDING = 2
   };

   struct buf {
       enum rwop op;
       int last;
       struct aiocb aiocb;
       unsigned char data[BSZ];
   };

   struct buf bufs[NBUF];
   unsigned char translate(unsigned char c) {
       if (isalpha(c)) {
           if (c >= 'n') {
               c -= 13;
           } else if (c >= 'a') {
               c += 13;
           } else if (c >= 'N') {
               c -= 13;
           } else {
               c += 13;
           }
       }
       return (c);
   }

   int main(int argc, char* argv[]) {
       int ifd, ofd, i, j, n, err, numop;
       struct stat sbuf;
       const struct aiocd* aiolist[NBUF];
       off_t off = 0;
       if (argc != 3) {
           err_quit("usage: rot13 infile outfile");
       }
       if ((ifd = open(argv[1], O_RDONLY)) < 0) {
           err_sys("can't open %s", argv[1]);
       }
       if ((ofd = open(argv[2], O_RDWR|O_CREAT|O_TRUNC, FILE_MODE)) < 0) {
           err_sys("can't create %s", argv[2]);
       }
       if (fstat(ifd, &sbuf) < 0) {
           err_sys("fstat error");
       }
       for (i = 0; i < NBUF; i++) {
           bufs[i].op = UNUSED;
           bufs[i].aiocb.aio_buf = bufs[i].data;
           bufs[i].aiocb.aio_sigevent.sigev_notify = SIGEV_NONE;
           aiolist[i] = NULL;
       }
       numop = 0;
       for ( ; ; ) {
           for (i = 0; i < NBUF; i++) {
               switch (bufs[i].op) {
                   case UNUSED:
                       if (off < sbuf.st_size) {
                           bufs[i].op = READ_PENDING;
                           bufs[i].aiocb.aio_fildes = ifd;
                           bufs[i].aiocb.aio_offset = off;
                           off += BSZ;
                           if (off >= sbuf.st_size) {
                               bufs[i].last = 1;
                           }
                           bufs[i].aiocb.aio_nbytes = BSZ;
                           if (aio_read(&bufs[i].aiocb) < 0) {
                               err_sys("aio_read failed");
                           }
                           aiolist[i] = &bufs[i].aiocb;
                           numop++;
                       }
                       break;
                   case READ_PENDING:
                       if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS) {
                           continue;
                       }
                       if (err != 0) {
                           if (err == -1) {
                               err_sys("aio_error failed");
                           } else {
                               err_exit(err, "read failed");
                           }
                       }
                       if ((n = aio_return(&bufs[i].aiocb)) < 0) {
                           err_sys("aio_return failed");
                       }
                       if (n != BSZ && !bufs[i].last) {
                           err_quit("short read (%d/%d)", n, BSZ);
                       }
                       for (j = 0; j < n; j++) {
                           bufs[i].data[j] = translate(bufs[i].data[j]);
                       }
                       bufs[i].op = WRITE_PENDING;
                       bufs[i].aiocb.aio_fildes = ofd;
                       bufs[i].aiocb.aio_nbytes = n;
                       if (aio_write(&bufs[i].aiocb) < 0) {
                           err_sys("aio_write failed");
                       }
                       break;
                   case WRITE_PENDING:
                       if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS) {
                           continue;
                       }
                       if (err != 0) {
                           if (err == -1) {
                               err_sys("aio_error failed");
                           } else {
                               err_exit(err, "write failed");
                           }
                       }
                       if ((n = aio_return(&bufs[i].aiocb)) < 0) {
                           err_sys("aio_return failed");
                       }
                       if (n != bufs[i].aiocb.aio_nbytes) {
                           err_quit("short write (%d/%d)", n, BSZ);
                       }
                       aiolist[i] = NULL;
                       bufs[i].op = UNUSED;
                       numop--;
                       break;
               }
           }
           if (numop == 0) {
               if (off >= sbuf.st_size) {
                   break;
               }
           } else {
               if (aio_suspend(aiolist, NBUF, NULL) < 0) {
                   err_sys("aio_suspend failed");
               }
           }
       }
       bufs[0].aiocb.aio_fildes = ofd;
       if (aio_fsync(O_SYNC, &bufs[0].aiocb) < 0) {
           err_sys("aio_fsync failed");
       }
       exit(0);
   }

readv和writev
~~~~~~~~~~~~~

readv和writev函数用于在一次函数调用中读、写多个非连续缓冲区。

这两个函数也被称为散布读（scatter read）和聚集写（gather write）。

.. code:: c

   #include <sys/aio.h>

   ssize_t readv(int fd, const struct iovec* iov, int iovcnt);
   ssize_t writev(int fd, const struct iovec* iov, int iovcnt);
   // 成功返回已读或已写的字节数，出错返回−1

其中struct iovec结构如下：

.. code:: c

   struct iovec {
       void* iov_base; /* starting address of buffer */
       size_t iov_len; /* size of buffer */
   };

iovcnt参数指定iov数组中的元素数。

writev返回输出的字节总数，通常应等于所有缓冲区长度之和。

readn和writen
~~~~~~~~~~~~~

管道、FIFO以及某些设备（特别是终端和网络）有以下两种性质：

1. 一次read操作所返回的数据可能少于所要求的数据，即使还没达到文件尾端也可能是这样。这不是一个错误，应当继续读该设备。

2. 一次write操作的返回值可能少于指定输出的字节数。这可能是由某个因素造成的，例如，内核输出缓冲区变满。这也不是错误，应当继续写余下的数据。

      通常，只有非阻塞描述符，或捕捉到一个信号时，才发生这种write的中途返回。

我们可以实现两个函数来读、写指定的N字节数据：

.. code:: c

   #include "apue.h"

   ssize_t readn(int fd, void* ptr, size_t n) {
       size_t nleft;
       ssize_t nread;
       nleft = n;
       while (nleft > 0) {
           if ((nread = read(fd, ptr, nleft)) < 0) {
               if (nleft == n) {
                   return -1;
               } else {
                   break;
               }
           }
           nleft -= nread;
           ptr += nread;
       }
       return n - nleft;
   }

   ssize_t writen(int fd, const void* ptr, size_t n) {
       size_t nleft;
       ssize_t nwritten;
       nleft = n;
       while (nleft > 0) {
           if ((nwritten = write(fd, ptr, nleft)) < 0) {
               if (nleft == n) {
                   return -1;
               } else {
                   break;
               }
           }
           nleft -= nwritten;
           ptr += nwritten;
       }
       return n - nleft;
   }

存储映射I/O
~~~~~~~~~~~

存储映射I/O（memory-mapped
I/O）能将一个磁盘文件映射到存储空间中的一个缓冲区上，于是，当从缓冲区中取数据时，就相当于读文件中的相应字节。将数据存入缓冲区时，相应字节就自动写入文件。这样，就可以在不使用read和write的情况下执行I/O。

我们可以使用mmap函数来告诉内核将一个给定的文件映射到一个存储区域中：

.. code:: c

   #include <sys/mman.h>

   void* mmap(void* addr, size_t len, int prot, int flag, int fd, off_t off);
   // 成功返回映射区的起始地址，出错返回MAP_FAILED

addr参数指定映射存储区的起始地址。通常设置为0，表示由系统选择该映射区的起始地址。

fd参数是指定要被映射文件的描述符。在文件映射到地址空间之前，必须先打开该文件。len参数是映射的字节数，off是要映射字节在文件中的起始偏移量。

prot参数指定了映射存储区的保护要求：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210511162827041.png
   :alt: image-20210511162827041

   image-20210511162827041

可将prot参数指定为PROT_NONE，也可指定为PROT_READ、PROT_WRITE和PROT_EXEC的任意组合的按位或。

flag参数的可能值：

-  MAP_FIXED：返回值必须等于addr。如果未指定此标志，而且addr非0，则内核只把addr视为在何处设置映射区的一种建议，但是不保证会使用所要求的地址。

-  MAP_SHARED：这一标志描述了本进程对映射区所进行的存储操作的配置。此标志指定存储操作修改映射文件，也就是，存储操作相当于对该文件的
   write。

-  MAP_PRIVATE：本标志说明，对映射区的存储操作导致创建该映射文件的一个私有副本。所有后来对该映射区的引用都是引用该副本。

      此标志的一种用途是用于调试程序，它将程序文件的正文部分映射至存储区，但允许用户修改其中的指令。任何修改只影响程序文件的副本，而不影响原文件。

必须指定MAP_SHARED和MAP_PRIVATE中的一个且只能指定一个。

off的值和addr的值（如果指定了MAP_FIXED）通常被要求是系统虚拟存储页长度的倍数。

信号SIGSEGV通常用于指示进程试图访问对它不可用的存储区。如果映射存储区被mmap指定成了只读的，那么进程试图将数据存入这个映射存储区的时候，也会产生此信号。

如果映射区的某个部分在访问时已不存在，则产生SIGBUS信号。

子进程能通过fork继承存储映射区（因为子进程复制父进程地址空间，而存储映射区是该地址空间中的一部分），但是由于同样的原因，新程序则不能通过exec继承存储映射区。

可以调用mprotect函数来更改一个现有映射的权限：

.. code:: c

   #include <sys/mman.h>

   int mprotect(void* addr, size_t len, int prot);
   // 成功返回0，出错返回-1

prot的合法值与mmap中prot参数的一样.

如果共享映射中的页已修改，那么可以调用msync将该页冲洗到被映射的文件中。msync函数类似于fsync，但作用于存储映射区。

.. code:: c

   #include <sys/mman.h>

   int msync(void* addr, size_t len, int flags);
   // 成功返回0，出错返回-1

如果映射是私有的，那么不修改被映射的文件。

flags参数控制如何冲洗存储区：

-  可以指定MS_ASYNC标志来简单地调试要写的页。
-  如果希望在返回之前等待写操作完成，可以指定MS_SYNC标志。
-  MS_INVALIDATE是一个可选标志，允许我们通知操作系统丢弃那些与底层存储器没有同步的页。

一定要指定MS_ASYNC和MS_SYNC中的一个。

当进程终止时，会自动解除存储映射区的映射，或者直接调用munmap函数也可以解除映射区。

.. code:: c

   #include <sys/mman.h>

   int munmap(void* addr, size_t len);
   // 成功返回0，出错返回-1

调用munmap并不会使映射区的内容写到磁盘文件上。对于MAP_SHARED区磁盘文件的更新，会在我们将数据写到存储映射区后的某个时刻，按内核虚拟存储算法自动进行。在存储区解除映射后，对MAP_PRIVATE存储区的修改会被丢弃。

   关闭映射存储区时使用的文件描述符并不解除映射区。

一个简单的示例(使用存储映射I/O复制文件)：

.. code:: c

   #include "apue.h"
   #include <fcntl.h>
   #include <sys/mman.h>

   #define COPYINCR (1024*1024*1024) // 1 GB

   int main(int argc, char *argv[]) {
       int fdin, fdout;
       void *src, *dst;
       size_t copysz;
       struct stat sbuf;
       off_t fsz = 0;
       if (argc != 3) {
           err_quit("usage: %s <fromfile> <tofile>", argv[0]);
       }
       if ((fdin = open(argv[1], O_RDONLY)) < 0) {
           err_sys("can't open %s for reading", argv[1]);
       }
       if ((fdout = open(argv[2], O_RDWR | O_CREAT | O_TRUNC, FILE_MODE)) < 0) {
           err_sys("can't open %s for writing", argv[2]);
       }
       if (fstat(fdin, &sbuf) < 0) {
           err_sys("fstat error");
       }
       if (ftruncate(fdout, sbuf.st_size) < 0) {
           err_sys("ftruncate error");
       }
       while (fsz < sbuf.st_size) {
           if ((sbuf.st_size - fsz) > COPYINCR) {
               copysz = COPYINCR;
           } else {
               copysz = sbuf.st_size - fsz;
           }
           if ((src = mmap(0, copysz, PROT_READ, MAP_SHARED, fdin, fsz)) == MAP_FAILED) {
               err_sys("mmap error for input");
           }
           if ((dst = mmap(0, copysz, PROT_READ | PROT_WRITE, MAP_SHARED, fdout, fsz)) == MAP_FAILED) {
               err_sys("mmap error for output");
           }
           memcpy(dst, src, copysz);
           munmap(src, copysz);
           munmap(dst, copysz);
           fsz += copysz;
       }
       exit(0);
   }
