文件I/O
-------

对于内核而言，所有打开的文件都通过文件描述符引用。文件描述符是一个非负整数。当打开一个现有文件或创建一个新文件时，内核向进程返回一个文件描述符。

open和openat
~~~~~~~~~~~~

调用open和openat可以打开或创建一个文件。

.. code:: c

   #include <fcntl.h>

   int open(const char* path, int oflag, ... /* mode_t mode */);
   int openat(int fd, const char* path, int oflag, ... /* mode_t mode */);
   // path参数是要打开或创建文件的名字，oflag参数可用来说明此参数的多个选项
   // 成功返回文件描述符，出错返回-1

oflag参数有：

-  ``O_RDONLY``\ 、\ ``O_WRONLY``\ 、\ ``O_RDWR``\ 、\ ``O_EXEC``\ (只执行打开)、\ ``O_SEARCH``\ (只搜索打开，应用于目录)，这五个常量\ **必须且只能指定一个**\ 。
-  ``O_APPEND``\ 、\ ``O_TRUNC``\ (截断)、\ ``O_CREAT``\ (不存在则创建)、\ ``O_SYNC``\ (使每次的write等待物理I/O完成，包括由该write操作引起的文件属性更新所需的I/O)、\ ``O_DSYNC``\ (使每次write要等待物理I/O操作完成，但是如果该写操作并不影响读取刚写入的数据，则不需等待文件属性被更新)。
-  更多内容请参考APUE第三版P50-P51。

由open和openat函数返回的文件描述符一定是\ **最小的未用描述符数值**\ 。

fd参数把open和openat函数区分开，共有3种可能性：

1. path参数指定的是绝对路径名，在这种情况下，fd参数被忽略，openat函数就相当于open函数。
2. path参数指定的是相对路径名，fd参数指出了相对路径名在文件系统中的开始地址。fd参数是通过打开相对路径名所在的目录来获取。
3. path参数指定了相对路径名，fd参数具有特殊值AT_FDCWD。在这种情况下，路径名在当前工作目录中获取，openat函数在操作上与open函数类似。

openat函数的作用：

1. 让线程可以使用相对路径名打开目录中的文件，而不再只能打开当前工作目录。\ **同一进程中的所有线程共享相同的当前工作目录**\ ，因此很难让同一进程的多个不同线程在同一时间工作在不同的目录中。
2. 以避免time-of-check-to-time-of-use（TOCTTOU）错误。

..

   TOCTTOU错误的基本思想是：如果有两个基于文件的函数调用，其中第二个调用依赖于第一个调用的结果，那么程序是脆弱的。

creat
~~~~~

可以使用creat函数创建一个新文件。

.. code:: c

   #include <fcntl.h>

   int creat(const char* path, mode_t mode);
   // 成功返回*只写打开*的文件描述符，出错返回-1
   // 等效于如下open函数调用
   open(path, O_WRONLY | O_CREAT | O_TRUNC, mode);

close
~~~~~

可以使用close函数关闭一个打开文件。

.. code:: c

   #include <unistd.h>

   int close(int fd);
   // 成功返回0，出错返回-1

关闭一个文件时还会释放该进程加在该文件上的所有记录锁。

**当一个进程终止时，内核自动关闭它所有的打开文件**\ 。很多程序都利用了这一功能而不显式地用close关闭打开文件。

lseek
~~~~~

可以使用lseek函数来设置文件偏移量。

.. code:: c

   #include <unistd.h>

   off_t lseek(int fd, off_t offset, int whence);
   // 若成功，返回新的文件偏移量，出错则返回-1

对参数offset的解释与参数whence的值有关：

-  若whence是SEEK_SET，则将该文件的偏移量设置为距文件开始处offset个字节。
-  若whence是SEEK_CUR，则将该文件的偏移量设置为其当前值加offset，offset可为正或负。
-  若whence是SEEK_END，则将该文件的偏移量设置为文件长度加offset，offset可正可负。

查看当前偏移量：

.. code:: c

   off_t currpos;
   currpos = lseek(fd, 0, SEEK_CUR);

read
~~~~

调用read函数从打开文件中读取数据。

.. code:: c

   #include <unistd.h>

   ssize_t read(int fd, void* buf, size_t nbytes);
   // 返回读到的字节数，若已经读到文件尾，则返回0，出错返回-1

write
~~~~~

调用write函数向打开文件中写数据。

.. code:: c

   #include <unistd.h>

   ssize_t write(int fd, const void* buf, size_t nbytes);
   // 返回已写的字节数，出错则返回-1

文件共享
~~~~~~~~

UNIX系统支持在不同进程间共享打开文件。

内核使用3种数据结构表示打开文件：

1. 每个进程在进程表中都有一个记录项，记录项中包含一张\ **打开文件描述符表**\ ，可将其视为一个矢量，每个描述符占用一项。与每个文件描述符相关联的是：

   -  文件描述符标志；
   -  指向一个文件表项的指针。

2. 内核为所有打开文件维持一张\ **文件表**\ 。每个文件表项包含：

   -  文件状态标志（读、写、添写、同步和非阻塞等）；
   -  当前文件偏移量；
   -  指向该文件v节点表项的指针。

3. 每个打开文件（或设备）都有一个 v
   节点（v-node）结构。v节点包含了文件类型和对此文件进行各种操作函数的指针。对于大多数文件，v节点还包含了该文件的i节点（i-node，索引节点）。i
   节点包含了文件的所有者、文件长度、指向文件实际数据块在磁盘上所在位置的指针等。

..

   Linux没有使用v节点，而是使用了通用i节点结构。虽然两种实现有所不同，但在概念上，v节点与i节点是一样的。两者都指向文件系统特有的i节点结构。

对前面操作的一些说明：

-  在完成每个write后，在文件表项中的当前文件偏移量即增加所写入的字节数。如果这导致当前文件偏移量超出了当前文件长度，则将i节点表项中的当前文件长度设置为当前文件偏移量。
-  如果用O_APPEND标志打开一个文件，则相应标志也被设置到文件表项的文件状态标志中。每次对这种具有追加写标志的文件执行写操作时，文件表项中的当前文件偏移量首先会被设置为i节点表项中的文件长度。这就使得每次写入的数据都追加到文件的当前尾端处。
-  若一个文件用lseek定位到文件当前的尾端，则文件表项中的当前文件偏移量被设置为i节点表项中的当前文件长度。
-  lseek函数只修改文件表项中的当前文件偏移量，不进行任何I/O操作。

原子操作
~~~~~~~~

一般而言，原子操作（atomic
operation）指的是由多步组成的一个操作。如果该操作原子地执行，则要么执行完所有步骤，要么一步也不执行，不可能只执行所有步骤的一个子集。

pread和pwrite可以原子性的定位并执行I/O(可用于多线程环境)：

.. code:: c

   #include <unistd.h>

   ssize_t pread(int fd, void* buf, size_t nbytes, off_t offset);
   // 返回读到的字节数，若已到达文件尾则返回0，出错返回-1
   ssize_t pwrite(int fd, const void* buf, size_t nbytes, off_t offset);
   // 成功返回读到的字节数，出错返回-1

dup与dup2
~~~~~~~~~

可以通过dup和dup2函数来复制一个现有的文件描述符。

.. code:: c

   #include <unistd.h>

   int dup(int fd);
   int dup2(int fd, int fd2);
   // 成功则返回新的文件描述符，出错则返回-1

   // 其中dup等效于
   fcntl(fd, F_DUPFD, 0); 
   // dup2等效于：
   close(fd2);
   fcntl(fd, F_DUPFD, fd2);
   // 不过dup2是原子的，而上述函数并不是

由dup返回的新文件描述符一定是当前可用文件描述符中的最小数值。

对于dup2，可以用fd2参数指定新描述符的值：

-  如果fd2已经打开，则先将其关闭。
-  如若fd等于fd2，则dup2返回fd2，而不关闭它。
-  否则，fd2的FD_CLOEXEC文件描述符标志就被清除，这样fd2在进程调用exec时是打开状态。

sync、fsync和fdatasync
~~~~~~~~~~~~~~~~~~~~~~

可以使用sync、fsync和fdatasync三个函数来保证磁盘上实际文件系统与缓冲区中内容的一致性。

.. code:: c

   #include <unistd.h>

   int fsync(int fd);
   int fdatasync(int fd);
   // 以上两个函数若成功则返回0，失败则返回-1
   void sync(void);

sync只是将\ **所有修改过**\ 的块缓冲区排入写队列，然后就返回，它并不等待实际写磁盘操作结束。

   通常，称为update的系统守护进程周期性地调用（一般每隔30秒）sync函数。这就保证了定期冲洗（flush）内核的块缓冲区。命令sync(1)也调用sync函数。

fsync函数只对由\ **文件描述符fd指定的一个文件**\ 起作用，并且等待写磁盘操作结束才返回。

   fsync可用于数据库这样的应用程序，这种应用程序需要确保修改过的块立即写到磁盘上。

fdatasync函数类似于fsync，但它只影响文件的数据部分。而除数据外，fsync还会同步更新文件的属性。

fcntl
~~~~~

fcntl函数可以改变已经打开文件的属性。

.. code:: c

   #include <fcntl.h>

   int fcntl(int fd, int cmd, ... /* int arg */);
   // 成功的返回值依赖于cmd，出错返回-1

fcntl函数有以下5种功能：

1. 复制一个已有的描述符（cmd=F_DUPFD或F_DUPFD_CLOEXEC）。
2. 获取/设置文件描述符标志（cmd=F_GETFD或F_SETFD）。
3. 获取/设置文件状态标志（cmd=F_GETFL或F_SETFL）。
4. 获取/设置异步I/O所有权（cmd=F_GETOWN或F_SETOWN）。
5. 获取/设置记录锁（cmd=F_GETLK、F_SETLK或F_SETLKW）。

查看状态标志时需要结合屏蔽字\ ``O_ACCMODE``\ ，使用方式可参考下方代码：

.. code:: c

   #include "apue.h"
   #include <fcntl.h>

   // the function of 3-12
   // 为文件描述符fd添加flags标志
   void set_fl(int fd, int flags) {
       int val;
       if ((val = fcntl(fd, F_GETFL, 0)) < 0) {
           err_sys("fcntl F_GETFL error");
       }
       val |= flags;
       if (fcntl(fd, F_SETFL, val) < 0) {
           err_sys("fcntl F_SETFL error");
       }
   }

   int main(int argc, char* argv[]) {
       int val;
       if (argc != 2) {
           err_quit("usage: %s <descriptor>", argv[0]);
       }
       if ((val = fcntl(atoi(argv[1]), F_GETFL, 0)) < 0) {
           err_sys("fcntl error for fd %d", atoi(argv[1]));
       }
       switch (val & O_ACCMODE) {
           case O_RDONLY:
               printf("read only");
               break;
           case O_WRONLY:
               printf("write only");
           case O_RDWR:
               printf("read write");
               break;
           default:
               err_dump("unknown access mode");
       }
       if (val & O_APPEND) {
           printf(", append");
       }
       if (val & O_NONBLOCK) {
           printf(", nonblockint");
       }
       if (val & O_SYNC) {
           printf("synchronous writes");
       }
   #if !defined(_POSIX_C_SOURCE) && defined(O_FSYNC) && (O_FSYNC != O_SYNC)
       if (val & O_FSYNC) {
           printf(", synchronous writes");
       }
   #endif
       putchar('\n');
       exit(0);
   }

程序运行时，设置O_SYNC标志会增加系统时间和时钟时间。

ioctl
~~~~~

ioctl是I/O操作的杂物箱，终端I/O是使用ioctl最多的地方。

.. code:: c

   #include <unistd.h>
   #include <sys/ioctl.h>

   int ioctl(int fd, int request, ...);
   // 若出错则返回-1，成功返回其他值

/dev/fd
~~~~~~~

较新的系统都提供名为/dev/fd 的目录，其目录项是名为 0、1、2
等的文件。打开文件/dev/fd/n等效于复制描述符n（假定描述符n是打开的）。
