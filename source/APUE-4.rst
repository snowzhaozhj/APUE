文件和目录
----------

stat、fstat、fstatat、lstat
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c

   #include <sys/stat.h>

   int stat(const char* restric pathname, struct stat* restrict buf);
   int fstat(inf fd, struct stat* buf);
   int lstat(const char* restrict pathname, strcut stat* restrict buf);
   int fstatat(int fd, const char* restrict pathname, struct stat* restrict buf, int flag);
   // 若成功则返回0，否则返回-1

lstat函数类似于stat，但是当命名的文件是一个符号链接时，lstat返回该符号链接的有关信息，而不是由该符号链接引用的文件的信息。

fstatat函数为一个相对于当前打开目录（由fd参数指向）的路径名返回文件统计信息。

-  flag参数控制着是否跟随着一个符号链接。当AT_SYMLINK_NOFOLLOW标志被设置时，fstatat不会跟随符号链接，而是返回符号链接本身的信息。否则，在默认情况下，返回的是符号链接所指向的实际文件的信息。

-  如果fd参数的值是AT_FDCWD，并且pathname参数是一个相对路径名，fstatat会计算相对于当前目录的pathname参数。

-  如果pathname是一个绝对路径，fd参数就会被忽略。

这后面两种情况下，根据flag的取值，fstatat的作用就跟stat或lstat一样。

   备注：后续有at的函数，规则都和这个差不多。

``struct stat``\ 结构的基本形式：

.. code:: c

   struct stat {
       mode_t st_mode;             /* file type & mode (permissions) */
       ino_t st_ino;               /* i-node number (serial number) */
       dev_t st_dev;               /* device number (file system) */
       dev_t st_rdev;              /* device number for special files */
       nlink_t st_nlink;           /* number of links */
       uid_t   st_uid;             /* user ID of owner */
       gid_t   st_gid;             /* group ID of owner */
       off_t   st_size;            /* size in bytes, for regular files */
       struct timespec st_atime;   /* time of last access */
       struct timespec st_mtime;   /* time of last modification */
       struct timespec st_ctime;   /* time of last file status change */
       blksize_t   st_blksize;     /* best I/O block size */
       blkcnt_t    st_blocks;      /* number of disk blocks allocated */
   };

文件类型
~~~~~~~~

1. 普通文件（regular
   file）。最常用的文件类型，这种文件包含了某种形式的数据。至于这种数据是文本还是二进制数据，对于UNIX内核而言并无区别。对普通文件内容的解释由处理该文件的应用程序进行。
2. 目录文件（directory
   file）。这种文件包含了其他文件的名字以及指向与这些文件有关信息的指针。对一个目录文件具有读权限的任一进程都可以读该目录的内容，但\ **只有内核可以直接写目录文件**\ 。
3. 块特殊文件（block special
   file）。这种类型的文件提供对设备（如磁盘）\ **带缓冲**\ 的访问，每次访问以固定长度为单位进行。
4. 字符特殊文件（character special
   file）。这种类型的文件提供对设备\ **不带缓冲**\ 的访问，每次访问长度可变。系统中的所有设备要么是字符特殊文件，要么是块特殊文件。
5. FIFO。这种类型的文件用于进程间通信，有时也称为命名管道（named
   pipe）。
6. 套接字（socket）。这种类型的文件用于进程间的网络通信。套接字也可用于在一台宿主机上进程之间的非网络通信。
7. 符号链接（symbolic link）。这种类型的文件指向另一个文件。

一个判断文件类型的程序：

.. code:: c

   #include "apue.h"

   int main(int argc, char *argv[]) {
       struct stat buf;
       char *ptr;
       for (int i = 1; i < argc; i++) {
           printf("%s: ", argv[i]);
           if (lstat(argv[i], &buf) < 0) {
               err_ret("lstat error");
               continue;
           }
           if (S_ISREG(buf.st_mode))
               ptr = "regular";
           else if (S_ISDIR(buf.st_mode))
               ptr = "directory";
           else if (S_ISCHR(buf.st_mode))
               ptr = "character special";
           else if (S_ISBLK(buf.st_mode))
               ptr = "block special";
           else if (S_ISFIFO(buf.st_mode))
               ptr = "fifo";
           else if (S_ISLNK(buf.st_mode))
               ptr = "symbolic link";
           else if (S_ISSOCK(buf.st_mode))
               ptr = "socket";
           else
               ptr = "** unknown mode **";
           printf("%s\n", ptr);
       }
       exit(0);
   }

设置用户ID和设置组ID
~~~~~~~~~~~~~~~~~~~~

当执行一个程序文件时，进程的有效用户ID通常就是实际用户ID，有效组ID通常是实际组ID。

可以在文件模式字（st_mode）中设置一个特殊标志，其含义是“当执行此文件时，将进程的有效用户ID设置为文件所有者的用户ID（st_uid）”。

还可以在文件模式字中可以设置另一位，它将执行此文件的进程的有效组ID设置为文件的组所有者ID（st_gid）。

在文件模式字中的这两位被称为\ **设置用户ID（set-user-ID）位**\ 和\ **设置组ID（set-group-ID）位**\ 。

这两位包含在文件的st_mode值中，分别可用常量\ ``S_ISUID``\ 和\ ``S_ISGID``\ 进行测试。

文件访问权限
~~~~~~~~~~~~

-  ``S_IRUSR``: 用户读，\ ``S_IWUSR``: 用户写，\ ``S_IXUSR``: 用户执行。

-  ``S_IRGRP``\ 、\ ``S_IWGRP``\ 、\ ``S_IXGRP``\ (组)

-  ``S_IROTH``\ 、\ ``S_IWOTH``\ 、\ ``S_IXOTH``\ (其他)

权限的使用规则：

-  我们用名字打开任一类型的文件时，对该名字中包含的每一个目录，包括它可能隐含的当前工作目录都应具有执行权限。（目录其执行权限位常被称为搜索位）
-  对于一个文件的读权限决定了我们是否能够打开现有文件进行读操作。这与open函数的O_RDONLY和O_RDWR标志相关。
-  对于一个文件的写权限决定了我们是否能够打开现有文件进行写操作。这与open函数的O_WRONLY和O_RDWR标志相关。
-  为了在open函数中对一个文件指定O_TRUNC标志，必须对该文件具有写权限。
-  为了在一个目录中创建一个新文件，必须对该目录具有写权限和执行权限。
-  为了删除一个现有文件，必须对包含该文件的目录具有写权限和执行权限。对该文件本身则不需要有读、写权限。
-  如果用7个exec函数中的任何一个执行某个文件，都必须对该文件具有执行权限。该文件还必须是一个普通文件。

进程每次打开、创建或删除一个文件时，内核就进行文件访问权限测试：

-  若进程的有效用户ID是0（超级用户），则允许访问。
-  若进程的有效用户ID等于文件的所有者ID（也就是进程拥有此文件），那么如果所有者适当的访问权限位被设置，则允许访问；否则拒绝访问。
-  若进程的有效组ID或进程的附属组ID之一等于文件的组ID，那么如果组适当的访问权限位被设置，则允许访问；否则拒绝访问。
-  若其他用户适当的访问权限位被设置，则允许访问；否则拒绝访问。

新文件和目录的所有权
~~~~~~~~~~~~~~~~~~~~

新文件的用户ID设置为进程的有效用户ID。关于组ID，POSIX.1允许实现选择下列之一作为新文件的组ID：

1. 新文件的组ID可以是进程的有效组ID。

2. 新文件的组ID可以是它所在目录的组ID。

access和faccessat
~~~~~~~~~~~~~~~~~

可以使用\ ``access``\ 函数来按照实际用户ID和实际组ID来进行访问权限测试。

.. code:: c

   #include <unistd.h>

   int access(const char* pathname, int mode);
   int faccessat(int fd, const char* pathname, int mode, int flag);
   // 成功返回0，出错返回-1

``mode``\ 可为：\ ``R_OK``\ 、\ ``W_OK``\ 、\ ``X_OK``\ ，分别测试读、写、执行权限。

accessat函数与access函数在下面两种情况下是相同的：

-  pathname为绝对路径；
-  fd参数取值为AT_FDCWD而pathname参数为相对路径。

否则，faccessat计算相对于打开目录（由fd参数指向）的pathname。

flag参数可以用于改变faccessat的行为，如果flag设置为AT_EACCESS，访问检查用的是调用进程的有效用户ID和有效组ID，而不是实际用户ID和实际组ID。

umask
~~~~~

umask函数为进程设置文件模式创建屏蔽字，并返回之前的值。

.. code:: c

   #include <sys/stat.h>
   mode_t umask(mode_t cmask);

umask值表示成八进制数，一位代表一种要屏蔽的权限。

chmod, fchmod, fchmodat
~~~~~~~~~~~~~~~~~~~~~~~

我们可以使用chmod函数来更改现有文件的权限。

.. code:: c

   #include <sys/stat.h>

   int chmod(const char* pathname, mode_t);
   int fchmod(int fd, mode_t mode);
   int fchmodat(int fd, const char* pathname, mode_t mode, int flag);

``fchmodat``\ 和\ ``chmod``\ 函数的区别可参考\ ``fstatat``\ 和\ ``stat``\ 的区别。

使用示例：

.. code:: c

   #include "apue.h"

   int main(int argc, char **argv) {
       struct stat statbuf;
       if (stat("foo", &statbuf) < 0) {
           err_sys("stat error for foo");
       }
       // 打开set-group-ID，关闭group-execute
       if (chmod("foo", (statbuf.st_mode & ~S_IXGRP) | S_ISGID) < 0) {
           err_sys("chmod error for foo");
       }
       if (chmod("bar", S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH) < 0) {
           err_sys("chmod error for bar");
       }
       return 0;
   }

chmod函数在下列条件下自动清除两个权限位：

-  Solaris等系统对用于普通文件的粘着位赋予了特殊含义，在这些系统上如果我们试图设置普通文件的粘着位（S_ISVTX），而且又没有超级用户权限，那么mode中的\ **粘着位**\ 自动被关闭。

-  如果新文件的组ID不等于进程的有效组ID或者进程附属组ID中的一个，而且进程没有超级用户权限，那么\ **设置组ID位**\ 会被自动被关闭。

粘着位
~~~~~~

粘着位(``S_ISVTX``)原本用于\ **保存正文**\ ，以提高效率。现今的系统扩展了粘着位的使用范围，如果对一个目录设置了粘着位，只有对该目录具有写权限的用户并且满足下列条件之一，才能删除或重命名该目录下的文件：

-  拥有此文件；
-  拥有此目录；
-  是超级用户。

chown、fchown、fchownat、lchown
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以使用chown函数来修改文件的用户ID和组ID。

.. code:: c

   #include <unistd.h>

   int chown(const char* pathname, uid_t owner, gid_t group);
   int fchown(int fd, uid_t owner, gid_t group);
   int fchownat(int fd, const char* pathname, uid_t owner, gid_t group);
   int lchown(const char* pathname, uid_t owner, gid_t group);
   // 成功返回0，失败返回-1

函数之间的区别可参考\ ``stat``\ 系列函数。

文件长度
~~~~~~~~

stat结构成员st_size表示以字节为单位的文件的长度。此字段只对普通文件、目录文件和符号链接有意义。

-  对于普通文件，其文件长度可以是0，在开始读这种文件时，将得到文件结束（end-of-file）指示。
-  对于目录，文件长度通常是一个数（如16或512）的整倍数。
-  对于符号链接，文件长度是在文件名中的实际字节数。

现今，大多数现代的UNIX系统提供字段st_blksize和st_blocks。其中，第一个是对文件I/O较合适的块长度，第二个是所分配的实际512字节块块数。

文件截断
~~~~~~~~

可以使用truncate函数来截断文件到指定长度length。

.. code:: c

   #include <unistd.h>

   int truncate(const char* pathname, off_t length);
   int ftruncate(int fd, off_t length);
   // 成功返回0，失败返回-1

文件系统
~~~~~~~~

我们可以把一个磁盘分成一个或多个分区。每个分区可以包含一个文件系统。i节点是固定长度的记录项，它包含有关文件的大部分信息。

|image0|

-  每个i节点中都有一个链接计数，其值是指向该i节点的目录项数。\ **只有当链接计数减少至0时，才可删除该文件**\ 。链接计数包含在\ ``stat``\ 结构的\ ``nlink_t``\ 成员中。这种链接类型被称为\ **硬链接**\ 。
-  另外一种链接类型称为\ **符号链接**\ （symbolic
   link）。符号链接文件的实际内容（在数据块中）包含了该符号链接所指向的文件的名字。
-  i节点包含了文件有关的所有信息：文件类型、文件访问权限位、文件长度和指向文件数据块的指针等。stat结构中的大多数信息都取自i节点。只有两项重要数据存放在目录项中：文件名和i节点编号。
-  因为目录项中的i节点编号指向同一文件系统中的相应i节点，一个目录项不能指向另一个文件系统的i节点。
-  当在不更换文件系统的情况下为一个文件重命名时，该文件的实际内容并未移动，只需构造一个指向现有i节点的新目录项，并删除老的目录项。链接计数不会改变。

link、linkat、unlink、unlinkat、remove
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c

   #include <unistd.h>

   int link(const char* existingpath, const char* newpath);
   int linkat(int efd, const char* existingpath, int nfd, const char* newpath, int flag);
   // 成功返回0，失败返回-1
   // 两者的区别可参考stat和fstatat
   // 这两个函数创建一个新目录项newpath，它引用现有文件existingpath。如果newpath已经存在，则返回出错。只创建newpath中的最后一个分量，路径中的其他部分应当已经存在。

   int unlink(const char* pathname);
   int unlinkat(int fd, const char* pathname, int flag);
   // 成功返回0，出错返回-1
   // 这两个函数解除对文件的链接

   #include <stdio.h>

   int remove(const char* pathname);
   // 成功返回0，出错返回-1
   // 对于文件，remove的作用与unlink相同，对于目录，remove的作用与rmdir相同

rename和renameat
~~~~~~~~~~~~~~~~

可以使用rename函数来重命名文件和目录。

.. code:: c

   #include <stdio.h>

   int rename(const char* oldname, const char* newname);
   int renameat(int oldfd, const char* oldname, int newfd, const char* newname);

重命名的情况：

-  如果oldname指的是一个文件而不是目录，那么为该文件或符号链接重命名。在这种情况下，如果newname已存在，则它不能引用一个目录。如果newname已存在，而且不是一个目录，则先将该目录项删除然后将oldname重命名为newname。对包含oldname的目录以及包含newname的目录，调用进程必须具有写权限，因为将更改这两个目录。
-  如若oldname指的是一个目录，那么为该目录重命名。如果newname已存在，则它必须引用一个目录，而且该目录应当是空目录。如果newname存在（而且是一个空目录），则先将其删除，然后将oldname重命名为newname。另外，当为一个目录重命名时，newname不能包含oldname作为其路径前缀。例如，不能将/usr/foo重命名为/usr/foo/testdir，因为旧名字（/usr/foo）是新名字的路径前缀，因而不能将其删除。
-  如若oldname或newname引用符号链接，则处理的是\ **符号链接本身**\ ，而不是它所引用的文件。
-  不能对.和..重命名。更确切地说，.和..都不能出现在oldname和newname的最后部分。
-  作为一个特例，如果oldname和newname引用同一文件，则函数不做任何更改而成功返回。

符号链接
~~~~~~~~

符号链接是对一个文件的\ **间接指针**\ ，它与硬链接有所不同，硬链接直接指向文件的i节点。引入符号链接的原因是为了避开硬链接的一些限制：

-  硬链接通常要求链接和文件位于同一文件系统中。

-  只有超级用户才能创建指向目录的硬链接（在底层文件系统支持的情况下）。

对符号链接以及它指向何种对象并无任何文件系统限制，任何用户都可以创建指向目录的符号链接。符号链接一般用于将一个文件或整个目录结构移到系统中另一个位置。

   运行ls命令，并使用\ ``-F``\ 选项时，符号链接后面会出现一个\ ``@``\ 符号

创建和读取符号链接
~~~~~~~~~~~~~~~~~~

可以使用symlink函数创建一个符号链接。

.. code:: c

   #include <unistd.h>

   int symlink(const char* actualpath, const char* sympath);
   int symlinkat(const char* actualpath, const char* sympath);
   // 成功返回0，出错返回-1
   // 函数创建了一个指向actualpath的新目录项sympath。

readlink提供了读取符号链接本身内容的功能(不会像open一样跟随链接)。

.. code:: c

   #include <unistd.h>

   ssize_t readlink(const char* restrict pathname, char* restrict buf, size_t bufsize);
   ssize_t readlinkat(int fd, const char* restrict pathname, char* restrict buf. size_t bufsize);
   // 成功返回读取的字节数，出错返回-1

文件时间
~~~~~~~~

每个文件维护了3个时间字段：

-  ``st_atime``: 文件数据的最后访问时间
-  ``st_mtime``: 文件数据的最后更改时间
-  ``st_ctime``: i节点状态的最后更改时间

..

   ls -u选项按访问时间排序，-c选项则按状态更改时间排序。

futimens, utimensat, utimes
~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以使用以下函数修改一个文件的访问和修改时间。

.. code:: c

   #include <sys/stat.h>

   int futimens(int fd, const struct timespect times[2]);
   int utimensat(int fd, const char* pathm const struct timespec times[2], int flag);
   // 成功返回0，出错返回-1

时间戳的指定方式：

-  如果times参数是一个空指针，则访问时间和修改时间两者都设置为当前时间。
-  如果times参数指向两个timespec结构的数组，任一数组元素的tv_nsec字段的值为UTIME_NOW，相应的时间戳就设置为当前时间，忽略相应的tv_sec字段。
-  如果times参数指向两个timespec结构的数组，任一数组元素的tv_nsec字段的值为UTIME_OMIT，相应的时间戳保持不变，忽略相应的tv_sec字段。
-  如果 times 参数指向两个 timespec 结构的数组，且 tv_nsec
   字段的值为既不是UTIME_NOW 也不是
   UTIME_OMIT，在这种情况下，相应的时间戳设置为相应的 tv_sec
   和tv_nsec字段的值。

执行这些函数需要的权限：

-  如果times是一个空指针，或者任一tv_nsec字段设为UTIME_NOW，则进程的有效用户ID必须等于该文件的所有者ID；进程对该文件必须具有写权限，或者进程是一个超级用户进程。
-  如果 times 是非空指针，并且任一 tv_nsec 字段的值既不是 UTIME_NOW
   也不是UTIME_OMIT，则进程的有效用户ID必须等于该文件的所有者ID，或者进程必须是一个超级用户进程。
-  如果times是非空指针，并且两个tv_nsec字段的值都为UTIME_OMIT，就不执行任何的权限检查。

.. code:: c

   #include <sys/time.h>

   int utimes(const char* pathname, const struct timeval times[2]);
   // 成功返回0，出错返回-1
   // 其中timeval的结构如下：
   struct timeval {
       time_t tv_sec;  // seconds
       long tv_usec;   // microseconds
   };

mkdir, mkdirat, rmdir
~~~~~~~~~~~~~~~~~~~~~

使用mkdir函数创建目录，使用rmdir函数来删除一个\ **空目录**\ 。

.. code:: c

   #include <sys/stat.h>

   int mkdir(const char* pathname, mode_t mode);
   int mkdirat(int fd, const char* pathname, mode_t mode);
   int rmdir(const char* pathname);
   // 成功返回0，出错返回-1

读目录
~~~~~~

.. code:: c

   #include <dirent.h>

   DIR* opendir(const char* pathname);
   DIR* fdopendir(int fd); // 为什么不叫fopendir呢?奇怪。
   // 成功返回指针，出错返回NULL

   struct dirent* readdir(DIR* dp);
   // 成功返回指针，出错返回NULL

   void rewinddir(DIR* dp); // 重设目录读取的位置为开头位置
   int closedir(DIR* dp);
   // 成功返回0，出错返回-1

   long telldir(DIR* dp);
   // 返回与dp关联的目录中的当前位置

   void seekdir(DIR* dp, long loc);

chdir, fchdir, getcwd
~~~~~~~~~~~~~~~~~~~~~

进程可以调用chdir来改变当前工作目录。

.. code:: c

   #include <unistd.h>

   int chdir(const char* pathname);
   int fchdir(int fd);
   // 成功返回0，失败返回-1

因为当前工作目录是进程的一个属性，所以它只影响调用 chdir
的进程本身，而不影响其他进程。

每个程序运行在独立的进程中，shell
的当前工作目录并不会随着程序调用chdir而改变。由此可见，为了改变shell进程自己的工作目录，shell应当直接调用chdir函数，为此，cd命令内建在shell中。

可以使用getcwd来获取当前目录。

.. code:: c

   #include <unistd.h>

   char* getcwd(char* buf, size_t size);
   // 成功返回buf, 出错返回NULL

设备特殊文件
~~~~~~~~~~~~

st_dev和st_rdev：

-  每个文件系统所在的存储设备都由其主、次设备号表示。设备号所用的数据类型是基本系统数据类型dev_t。主设备号标识设备驱动程序，有时编码为与其通信的外设板；次设备号标识特定的子设备。

-  我们通常可以使用两个宏：major和minor来访问主、次设备号。

-  系统中与每个文件名关联的 st_dev
   值是文件系统的设备号，该文件系统包含了这一文件名以及与其对应的i节点。

-  只有字符特殊文件和块特殊文件才有st_rdev值。此值包含实际设备的设备号。

.. |image0| image:: https://gitee.com/snow_zhao/img/raw/master/img/Image00076.jpg
