UNIX基础知识
------------

   读后感:

   这一章是Unix的简介，作者用简练的语言总结了Unix的基础知识，感觉写的很清晰。

UNIX体系结构
~~~~~~~~~~~~

从严格意义上说，可将操作系统定义为一种软件，它控制计算机硬件资源，提供程序运行环境。我们通常将这种软件称为内核。

内核的结构被称为\ **系统调用**\ 。

广义上说，操作系统包括了内核和一些其他软件(系统实用程序（system
utility）、应用程序、shell以及公用函数库等)。

登录
~~~~

系统在\ ``/etc/passwd``\ 文件中存储了登录项，登录项由7个以冒号分隔的字段组成：

-  登录名

-  加密口令

-  数字用户ID

-  数字组ID

-  注释字段

-  起始目录

-  shell程序

..

   目前，所有的系统已将加密口令移动到另一个文件中。

文件与目录
~~~~~~~~~~

UNIX文件系统是目录和文件的一种层次结构，所有东西的起点是称为根（root）的目录，这个目录的名称是一个字符“/”。

目录（directory）是一个包含目录项的文件。

目录中的各个名字称为文件名。

创建新目录时会自动创建了两个文件名：.（称为点）和..（称为点点）。点指向当前目录，点点指向父目录。在最高层次的根目录中，点点与点相同。

由斜线分隔的一个或多个文件名组成的序列（也可以斜线开头）构成路径名（pathname），以斜线开头的路径名称为绝对路径名（absolute
pathname），否则称为相对路径名（relative
pathname）。相对路径名指向相对于当前目录的文件。

每个进程都有一个工作目录（working
directory），有时称其为当前工作目录（current working
directory）。所有相对路径名都从工作目录开始解释。进程可以用chdir函数更改其工作目录。

登录时，工作目录设置为起始目录（home
directory），该起始目录从口令文件中相应用户的登录项中取得。

输入和输出
~~~~~~~~~~

文件描述符（file
descriptor）通常是一个小的非负整数，内核用以标识一个特定进程正在访问的文件。当内核打开一个现有文件或创建一个新文件时，它都返回一个文件描述符。

每当运行一个新程序时，所有的 shell 都为其打开 3
个文件描述符，即标准输入（standard input）、标准输出（standard
output）以及标准错误（standard error）。

函数open、read、write、lseek以及close提供了不带缓冲的I/O。这些函数都使用文件描述符。

标准I/O函数为那些不带缓冲的I/O函数提供了一个带缓冲的接口。

程序和进程
~~~~~~~~~~

程序的执行实例被称为进程（process）。

UNIX系统确保每个进程都有一个唯一的数字标识符，称为进程ID（process
ID）。进程ID总是一个非负整数。

有3个用于进程控制的主要函数：fork、exec和waitpid。（exec函数有7种变体，经常把它们统称为exec函数。）

一个进程内的所有线程共享同一地址空间、文件描述符、栈以及与进程相关的属性。因为它们能访问同一存储区，所以各线程在访问共享数据时需要采取同步措施以避免不一致性。

与进程相同，线程也用ID标识。但是，线程ID只在它所属的进程内起作用。一个进程中的线程
ID 在另一个进程中没有意义。

出错处理
~~~~~~~~

UNIX系统函数出错时，通常会返回一个负值，而且整型变量errno通常被设置为具有特定信息的值。

而有些函数对于出错则使用另一种约定而不是返回负值。例如，大多数返回指向对象指针的函数，在出错时会返回一个null指针。

POSIX和ISO
C将errno定义为一个符号，它扩展成为一个可修改的整形左值（lvalue）。它可以是一个包含出错编号的整数，也可以是一个返回出错编号指针的函数。以前使用的定义是：

.. code:: c

   extern int errno;

但是在支持线程的环境中，多个线程共享进程地址空间，每个线程都有属于它自己的局部errno以避免一个线程干扰另一个线程。例如，Linux支持多线程存取errno，将其定义为：

.. code:: c

   extern int *__errno_location(void);
   #define errno (*__errno_location())

对于 errno 应当注意两条规则：

1. 如果没有出错，其值不会被例程清除。因此，仅当函数的返回值指明出错时，才检验其值。

2. 任何函数都不会将 errno
   值设置为0，而且在<errno.h>中定义的所有常量都不为0。

C标准中定义了两个函数，用于打印出错信息：

.. code:: c

   #include <string.h>
   char* strerror(int errnum); // 将errornum(通常就是errno值)映射为一个出错消息字符串，并返回指向该字符串的指针

   #include <stdio.h>
   void perror(const char *msg);   
   // 基于error的当前值，在标准错误上产生一条出错消息，然后返回
   // 首先输出msg指向的字符，然后输出冒号和一个空格，接着输出对应于errno值的出错消息，最后输出一个换行符
   // 通常我们以argv[0]作为参数传入，也即文件名

可将在<errno.h>中定义的各种出错分成两类：致命性的和非致命性的。

-  对于致命性的错误，无法执行恢复动作。最多能做的是在用户屏幕上打印出一条出错消息或者将一条出错消息写入日志文件中，然后退出。

-  对于非致命性的出错，有时可以较妥善地进行处理。大多数非致命性出错是暂时的（如资源短缺）。

用户标识
~~~~~~~~

口令文件登录项中的用户ID（user
ID）是一个数值，它向系统标识各个不同的用户。

用户 ID 为 0
的用户为根用户（root）或超级用户（superuser）。在口令文件中，通常有一个登录项，其登录名为
root，我们称这种用户的特权为超级用户特权。

口令文件登录项也包括用户的组ID（group
ID），它是一个数值。组ID也是由系统管理员在指定用户登录名时分配的。一般来说，在口令文件中有多个登录项具有相同的组
ID。组被用于将若干用户集合到项目或部门中去。这种机制允许同组的各个成员之间共享资源（如文件）。

组文件将组名映射为数值的组ID。组文件通常是/etc/group。

除了在口令文件中对一个登录名指定一个组ID外，大多数
UNIX系统版本还允许一个用户属于另外一些组。这一功能是从4.2BSD开始的，它允许一个用户属于多至16个其他的组。登录时，读文件/etc/group，寻找列有该用户作为其成员的前
16 个记录项就可以得到该用户的附属组ID（supplementary group ID）。

信号
~~~~

信号（signal）用于通知进程发生了某种情况。

进程有以下3种处理信号的方式：

1. 忽略信号。不推荐使用这种处理方式。

2. 按系统默认方式处理。对于除数为0，系统默认方式是终止该进程。

3. 提供一个函数，信号发生时调用该函数，这被称为捕捉该信号。通过提供自编的函数，我们就能知道什么时候产生了信号，并按期望的方式处理它。

时间值
~~~~~~

系统基本数据类型time_t用于保存UTC时间值(日历时间)。

系统基本数据类型clock_t用于保存CPU时间值(进程时间)。

UNIX系统为一个进程维护了3个进程时间值：

-  时钟时间；

-  用户CPU时间；

-  系统CPU时间。

**用户CPU时间**\ 是执行用户指令所用的时间量。

**系统CPU时间**\ 是为该进程执行内核程序所经历的时间。

用户CPU时间和系统CPU时间之和常被称为\ **CPU时间**\ 。

要取得进程的时钟时间、用户时间和系统时间可以执行命令time(1)。

系统调用和库函数
~~~~~~~~~~~~~~~~

所有的操作系统都提供多种服务的入口点，由此程序向内核请求服务。各种版本的UNIX实现都提供良好定义、数量有限、直接进入内核的入口点，这些入口点被称为\ **系统调用**\ 。

应用程序既可以调用系统调用也可以调用库函数。很多库函数则会调用系统调用。

系统调用通常提供一种最小接口，而库函数通常提供比较复杂的功能。
