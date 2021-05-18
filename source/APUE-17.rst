高级进程间通信
--------------

   本章偏向于实践，所以书中很多内容不记录到笔记中。我目前掌握这些代码还是有些困难的:sob:。

UNIX域套接字
~~~~~~~~~~~~

UNIX域套接字用于在同一台计算机上运行的进程之间的通信。虽然因特网域套接字可用于同一目的，但
UNIX域套接字的效率更高。

UNIX域套接字提供流和数据报两种接口。UNIX域数据报服务是可靠的，既不会丢失报文也不会传递出错。

可以使用socketpair函数来创建一对无命名的、相互连接的UNIX域套接字：

.. code:: c

   #include <sys/socket.h>

   int socketpair(int domain, int type, int protocol, int sockfd[2]);
   // 成功返回0，出错返回-1

一对相互连接的UNIX域套接字可以起到全双工管道的作用：两端对读和写开放。

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210513203943007.png
   :alt: image-20210513203943007

   image-20210513203943007

命名UNIX域套接字
~~~~~~~~~~~~~~~~

UNIX域套接字的地址由sockaddr_un结构表示。

该结构的格式和具体实现相关。其中Linux3.2.0的表示为：

.. code:: c

   struct sockaddr_un {
       sa_family_t sun_family; // AF_UNIX
       char sun_path[108];     // pathname
   };

sockaddr_un结构的sun_path成员包含一个路径名。当我们将一个地址绑定到一个UNIX域套接字时，系统会用该路径名创建一个S_IFSOCK类型的文件。

利用该地址和bind函数，我们就能创建一个命名的UNIX域套接字。

   该文件仅用于向客户进程告示套接字名字。该文件无法打开，也不能由应用程序用于通信。

如果我们试图绑定同一地址时，该文件已经存在，那么bind请求会失败。当关闭套接字时，并不自动删除该文件，所以必须确保在应用程序退出前，对该文件执行解除链接操作。

使用示例：

.. code:: c

   #include "apue.h"
   #include <sys/socket.h>
   #include <sys/un.h>

   int main() {
       int fd, size;
       struct sockaddr_un un;
       un.sun_family = AF_UNIX;
       strcpy(un.sun_path, "foo.socket");
       if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0) {
           err_sys("socket failed");
       }
       size = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);
       // offsetof is a macro: 
       // #define offsetof(TYPE, MEMBER) ((int)&((TYPE *)0)->MEMBER)
       if (bind(fd, (struct sockaddr*)&un, size) < 0) {
           err_sys("bind failed");
       }
       printf("UNIX domain socket bound\n");
       exit(0);
   }

getopt
~~~~~~

可以利用getopt函数来处理命令行选项。

之前看到过一篇写的还不错的\ `博客 <https://www.cnblogs.com/qingergege/p/5914218.html>`__\ ，可以一起参考。

.. code:: c

   #include <unistd.h>

   int getopt(int argc, char *const argv[], const char *options);
   // 若所有选项被处理完则返回-1，否则返回下一个选项字符
   extern int optind, opterr, optopt;
   extern char *optarg;

参数argc和argv与传入main函数的一样。

options参数是一个包含该命令支持的选项字符的字符串。如果一个选项字符后面接了一个冒号，则表示该选项需要参数；否则，该选项不需要额外参数。

当遇到无效的选项时，getopt返回一个问题标记（应该就是指\ ``?``\ 这个字符）而不是这个字符。如果选项缺少参数，getopt也会返回一个问题标记，但如果选项字符串的第一个字符是冒号，getopt会直接返回冒号。

特殊的“–”格式则会导致getopt停止处理选项并返回-1。这允许用户传递以“-”开头但不是选项的参数。

   例如，如果有一个名字为“-bar”的文件，下面的命令行是无法删除这个文件的：

   .. code:: sh

      rm –bar

   因为rm会试图把-bar解释为选项。正确的删除文件的命令应该是：

   .. code:: sh

      rm -- -bar

getopt函数支持以下4个外部变量。

-  optarg：如果一个选项需要参数，在处理该选项时，getopt会设置optarg指向该选项的参数字符串。

-  opterr：如果一个选项发生了错误，getopt会默认打印一条出错消息。应用程序可以通过设置opterr参数为0来禁止这个行为。

-  optind：用来存放下一个要处理的字符串在argv数组里的下标。它从1开始，每处理一个参数，getopt都会对其递增1。

-  optopt：如果处理选项时发生了错误，getopt会设置optopt指向导致出错的选项字符串。

截取的一部分示例程序：

.. code:: c

   while ((c = getopt(argc, argv, "d")) != EOF) {
       switch (c) {
           case "d":
               debug = 1;
               break;
           case '?':
               err_quit("unrecognized option -%c", optopt);
       }
   }

一些更详细的示例可以参考我刚刚分享的\ `博客 <https://www.cnblogs.com/qingergege/p/5914218.html>`__\ 。
