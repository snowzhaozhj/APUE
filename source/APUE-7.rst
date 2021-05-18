进程环境
--------

C程序总是从main函数开始执行。main函数的原型是：

.. code:: c

   int main(int argc, char *argv[]);

其中，argc是命令行参数的数目，argv是指向参数的各个指针所构成的数组。

当内核通过\ ``exec``\ 函数执行C程序时，在调用main前先调用一个特殊的启动例程。可执行程序文件将此启动例程指定为程序的起始地址（这是由连接编辑器设置的，而连接编辑器则由C编译器调用）。启动例程从内核取得命令行参数和环境变量值，为调用main函数做好准备。

进程终止
~~~~~~~~

有8种方式使进程终止（termination）。

其中5种为正常终止：

1. 从main返回；
2. 调用\ ``exit``\ ；
3. 调用\ ``_exit``\ 或\ ``_Exit``\ ；
4. 最后一个线程从其启动例程返回；
5. 从最后一个线程调用\ ``pthread_exit``\ 。

3种为异常终止：

6. 调用\ ``abort``\ ；
7. 接到一个信号；
8. 最后一个线程对取消请求做出响应。

可以使用exit函数来退出程序：

.. code:: c

   #include <stdlib.h>

   void exit(int status);
   void _Exit(int status);

   #include <unistd.h>

   void _exit(int status);

其中\ ``_exit``\ 和\ ``_Exit``\ 立即进入内核，\ ``exit``\ 则先执行一些清理处理，然后返回内核。

传入的\ ``status``\ 参数被称为\ **终止状态**\ （在shell中可以通过\ ``echo $?``\ 命令来查看，没记错的话，状态的最大值好像是255）。

main函数返回一个整型值与用该值调用exit是等价的。

可以通过\ ``atexit``\ 函数来注册\ **终止处理程序**\ ：

.. code:: c

   #include <stdlib.h>

   int atexit(void (*func)(void));
   // 成功返回0，失败返回1

一个进程最多可以注册32个终止处理程序，这些函数将由exit自动调用。exit调用这些函数的顺序与它们注册的顺序\ **相反**\ 。同一函数如若登记多次，也会被调用多次。

根据ISO
C和POSIX.1，exit首先调用各终止处理程序，然后关闭（通过fclose）所有打开流。POSIX.1扩展了ISO
C标准，它说明，如若程序调用exec函数族中的任一函数，则将清除所有已安装的终止处理程序。

atexit函数的使用示例：

.. code:: c

   #include "apue.h"

   static void my_exit1(void);
   static void my_exit2(void);

   int main() {
       if (atexit(my_exit2) != 0) {
           err_sys("can't register my_exit2");
       }
       if (atexit(my_exit1) != 0) {
           err_sys("can't unregister my_exit1");
       }
       if (atexit(my_exit1) != 0) {
           err_sys("can't unregister my_exit1");
       }
       printf("main is done\n");
       return 0;
   }

   static void my_exit1(void) {
       printf("first exit handler\n");
   }

   static void my_exit2(void) {
       printf("second exit handler\n");
   }

输出结果：

::

   main is done
   first exit handler
   first exit handler
   second exit handler

命令行参数
~~~~~~~~~~

ISO C和POSIX.1都要求argv[argc]是一个空指针。

环境表
~~~~~~

每个程序都接收到一张环境表。环境表是一个字符指针数组，其中每个指针包含一个以null结束的C字符串的地址。全局变量environ则包含了该指针数组的地址：

.. code:: c

   extern char **environ;

我们称environ为环境指针（environment
pointer），指针数组为环境表，其中各指针指向的字符串为环境字符串。

C程序的存储布局
~~~~~~~~~~~~~~~

C程序一直由下列几部分组成：

-  正文段。这是由CPU执行的机器指令部分。通常，正文段是可共享的，还是只读的。

-  初始化数据段。通常将此段称为数据段，它包含了程序中需明确地赋初值的变量。

-  未初始化数据段。通常将此段称为bss段，在程序开始执行之前，内核将此段中的数据初始化为0或空指针。

-  栈。自动变量以及每次函数调用时所需保存的信息都存放在此段中。每次函数调用时，其返回地址以及调用者的环境信息（如某些机器寄存器的值）都存放在栈中。

      递归函数每次调用自身时，就用一个新的栈帧，因此一次函数调用实例中的变量集不会影响另一次函数调用实例中的变量。

-  堆。通常在堆中进行动态存储分配。由于历史上形成的惯例，堆位于未初始化数据段和栈之间。

典型的存储空间布局：

可以通过size(1)命令来查看各个段的大小。

共享库
~~~~~~

共享库使得可执行文件中不再需要包含公用的库函数，而只需在所有进程都可引用的存储区中保存这种库例程的一个副本。

程序第一次执行或者第一次调用某个库函数时，用动态链接方法将程序与共享库函数相链接。

这减少了每个可执行文件的长度，但增加了一些运行时间开销。这种时间开销发生在该程序第一次被执行时，或者每个共享库函数第一次被调用时。

共享库的另一个优点是可以用库函数的新版本代替老版本而无需对使用该库的程序重新连接编辑（假定参数的数目和类型都没有发生改变）。

存储空间分配
~~~~~~~~~~~~

可以使用malloc、calloc、realloc函数来动态分配内存：

.. code:: c

   #include <stdlib.h>

   void* malloc(size_t size);
   void* calloc(size_t nobj, size_t size);
   void* realloc(void* ptr, size_t newsize);
   // 成功返回非空指针，出错返回NULL

这3个分配函数所返回的指针一定是\ **适当对齐**\ 的，使其可用于任何数据对象。

如果ptr为NULL，那么realloc的功能和malloc相同。

可以使用free函数来释放动态分配的内存：

.. code:: c

   #include <stdlib.h>

   void free(void* ptr);

环境变量
~~~~~~~~

环境字符串的格式是：

.. code:: sh

   name=value

可以使用getenv函数来获取环境变量值：

.. code:: c

   #include <stdlib.h>

   char* getenv(const char* name);
   // 找到则返回与name相关联的value的指针，没找到返回NULL

可以使用putenv, setenv, unsetenv来设置环境变量：

.. code:: c

   #include <stdlib.h>

   int putenv(char* str);
   // 成功返回0，出错返回非0
   int setenv(const char* name, const char* value, int rewrite);
   int unsetenv(const char* name);
   // 成功返回0，出错返回-1

-  putenv将形式为name=value的字符串放到环境表中。如果name已经存在，则覆盖原来的定义。
-  setenv将name设置为value。若rewrite非0，则已存在时会进行覆盖；否则不删除其现有定义，也不报错。
-  unsetenv删除name的定义。即使不存在这种定义也不算出错。

..

   putenv和setenv之间的差别：

   -  setenv必须分配存储空间，以便依据其参数创建name=value字符串。
   -  putenv可以自由地将传递给它的参数字符串直接放到环境中。

   因此，将存放在栈中的字符串作为参数传递给putenv就会发生错误，其原因是，从当前函数返回时，其栈帧占用的存储区可能将被重用。

setjmp和longjmp
~~~~~~~~~~~~~~~

.. code:: c

   #include <setjmp.h>

   int setjmp(jmp_buf env);
   // 直接调用返回0，从longjmp中返回，返回非0
   void longjmp(jmp_buf env, int val);

一个使用示例：

.. code:: c

   #include "apue.h"
   #include <setjmp.h>

   static void f1(int, int, int, int);
   static void f2(void);

   static jmp_buf jmpbuffer;
   static int globalval;

   int main() {
       int autoval;
       register int regival;
       volatile int volval;
       static int staval;
       globalval = 1, autoval = 2, regival = 3, volval = 4, staval = 5;
       if (setjmp(jmpbuffer) != 0) {
           printf("after longjmp:\n");
           printf("globalval = %d, autoval = %d, regival = %d, volval = %d, staval = %d\n", globalval, autoval, regival, volval, staval);
           exit(0);
       }
       globalval = 95, autoval = 96, regival = 97, volval = 98, staval = 99;
       f1(autoval, regival, volval, staval);
       return 0;
   }

   static void f1(int i, int j, int k, int l) {
       printf("in f1():\n");
       printf("globalval = %d, autoval = %d, regival = %d, volval = %d, staval = %d\n", globalval, i, j, k, l);
       f2();
   }

   static void f2(void) {
       longjmp(jmpbuffer, 1);
   }

输出结果：

::

   in f1():
   globalval = 95, autoval = 96, regival = 97, volval = 98, staval = 99
   after longjmp:
   globalval = 95, autoval = 96, regival = 3, volval = 98, staval = 99

变量在调用longjmp后值是否回滚是\ **不确定的**\ 。

如果你有一个自动变量，而又不想使其值回滚，则可定义其为具有volatile属性。

声明为全局变量或静态变量的值在执行longjmp时保持不变。

getrlimit和setrlimit
~~~~~~~~~~~~~~~~~~~~

每个进程都有一组资源限制，其中一些可以用getrlimit和setrlimit函数查询和更改。

.. code:: c

   #include <sys/resource.h>

   int getrlimit(int resource, struct rlimit* rlptr);
   int setrlimit(int resource, struct rlimit* rlptr);
   // 成功返回0，出错返回非0

其中\ ``struct rlimit``\ 的结构如下：

.. code:: c

   struct rlimit {
       rlim_t rlim_cur; /* soft limit: current limit */
       rlim_t rlim_max; /* hard limit: maximum value for rlim_cur */
   };

在更改资源限制时，须遵循下列3条规则：

1. 任何一个进程都可将一个软限制值更改为小于或等于其硬限制值。
2. 任何一个进程都可降低其硬限制值，但它必须大于或等于其软限制值。这种降低，对普通用户而言是不可逆的。
3. 只有超级用户进程可以提高硬限制值。

常量RLIM_INFINITY指定了一个无限量的限制。

resource参数的取值可参考APUE第三版P176-177。
