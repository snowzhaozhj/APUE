Unix标准及实现
--------------

   这一章前面一部分介绍了Unix操作系统的标准和实现，后面部分主要是对一些\ **宏**\ 的讨论。

   个人认为这一章对我这样的初级程序员来说没有那么重要，因此仅记录一点个人觉得值得一记的东西。

一些名称缩写
~~~~~~~~~~~~

``ANSI``: American National Standards Institute, 美国国家标准学会。

``ISO``: International Organization for Standardization,
国际标准化组织。

``IEC``: International Electrotechnical Commission, 国际电子技术协会。

``IEEE``: Institute of Electrical and Electronic Engineers,
电气和电子工程师学会。

``POSIX``: Portable Operating System Interface, 可移植操作系统接口。

``SUS``: Single UNIX Specification, 单一UNIX规范。

``XSI``: X/Open System Interface, X/Open系统接口。

限制
~~~~

限制可分为两种：编译时限制和运行时限制。

编译时限制可在头文件中定义，而运行时限制需要进程调用一个函数来获取限制值。

UNIX提供了以下三种限制：

1. 编译时限制（头文件）。
2. 与文件或目录无关的运行时限制（\ ``sysconf``\ 函数）。
3. 与文件或目录有关的运行时限制（\ ``pathconf``\ 和\ ``fpathconf``\ 函数）。

.. code:: c

   #include <unistd.h>
   long sysconf(int name);
   long pathconf(const char* pathname, int name);  // 使用路径名作为参数
   long fpathconf(int fd, int name);               // 使用文件描述符作为参数
   // 以上三个函数成功返回对应值，出错返回-1
   // 如果name参数并不是一个合适的常量，那么三个函数都返回-1，并把errno设置为EINVAL
   // 有些name会返回一个变量值(返回值>=0)或者提示该值是不确定的，不确定的值通过返回-1来体现，而不改变errno的值

选项
~~~~

对于每一个选项，有以下3种可能的平台支持状态。

1. 如果符号常量没有定义或者定义值为−1，那么该平台在编译时并不支持相应选项。

2. 如果符号常量的定义值大于0，那么该平台支持相应选项。

3. 如果符号常量的定义值为0，则必须调用sysconf、pathconf或fpathconf来判断相应选项是否受到支持。在这种情况下，这些函数的name参数前缀\ ``_POSIX``\ 必须替换为\ ``_SC``\ 或\ ``_PC``

      对于以\ ``_XOPEN``\ 为前缀的常量，在构成name参数时必须在其前放置\ ``_SC``\ 或\ ``_PC``\ 。例如，若常量\ ``_POSIX_RAW_THREADS``\ 是未定义的，那么就可以将name参数设置为SC_RAW_THREADS，并以此调用sysconf来判断该平台是否支持POSIX线程选项。如若常量\ ``_XOPEN_UNIX``\ 是未定义的，那么就可以将name参数设置为\ ``_SC_XOPEN_UNIX``\ ，并以此调用sysconf来判断该平台是否支持XSI扩展。
