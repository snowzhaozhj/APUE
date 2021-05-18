流和FILE对象
------------

对于标准I/O库，它们的操作是围绕流（stream）进行的。

标准I/O文件流可用于单字节或多字节（“宽”）字符集。

流的定向（stream’s
orientation）决定了所读、写的字符是单字节还是多字节的。

-  当一个流最初被创建时，它并没有定向。
-  如若在未定向的流上使用一个多字节I/O函数，则将该流的定向设置为宽定向的。
-  若在未定向的流上使用一个单字节I/O函数，则将该流的定向设为字节定向的。

只有两个函数可改变流的定向。freopen函数清除一个流的定向；fwide函数可用于设置流的定向。

.. code:: c

   #include <stdio.h>
   #include <wchar.h>

   int fwide(FILE* fp, int mode);
   // 若流是宽定向的，那么返回正值
   // 若流是字节定向的，返回负值
   // 若流是为定向的，那么返回0

根据mode参数的不同值，fwide函数执行不同的工作。

-  如若mode参数值为负，fwide将试图使指定的流是字节定向的。
-  如若mode参数值为正，fwide将试图使指定的流是宽定向的。
-  如若mode参数值为0，fwide将不试图设置流的定向，但返回标识该流定向的值。

fwide并不改变已定向流的定向。

当打开一个流时，标准I/O函数fopen返回一个指向FILE对象的指针。该对象通常是一个结构，它包含了标准I/O库为管理该流需要的所有信息，包括用于实际I/O的文件描述符、指向用于该流缓冲区的指针、缓冲区的长度、当前在缓冲区中的字符数以及出错标志等。

标准输入、标准输出、标准错误
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对一个进程预定义了三个流，这三个流可以自动的被进程使用，这3个标准I/O流通过预定义文件指针\ ``stdin``\ 、\ ``stdout``\ 和\ ``stderr``\ 加以引用（定义在头文件<stdio.h>中）。

缓冲
~~~~

标准I/O提供了以下3种类型的缓冲。

1. 全缓冲。填满标准I/O缓冲区后才进行实际I/O操作。对于驻留在磁盘上的文件通常是由标准I/O库实施全缓冲的。
2. 行缓冲。当在输入和输出中遇到换行符时，标准I/O库执行I/O操作。当流涉及一个终端时（如标准输入和标准输出），通常使用行缓冲。
3. 不带缓冲。标准I/O库不对字符进行缓冲存储。

..

   对于行缓冲有两个限制：

   -  只要填满了缓冲区，那么即使还没有写一个换行符，也进行I/O操作。
   -  任何时候只要通过标准I/O
      库要求从一个不带缓冲的流，或者一个行缓冲的流（它从内核请求需要数据）得到输入数据，那么就会冲洗所有行缓冲输出流。

标准错误流stderr通常是不带缓冲的，这就使得出错信息可以尽快显示出来，而不管它们是否含有一个换行符。

可以使用以下两个函数来更改缓冲类型：

.. code:: c

   #include <stdio.h>

   void setbuf(FILE* restrict fp, char* restric buf);
   int setvbuf(FILE* restrict fp, char* restric buf, int mode, size_t size);
   // 成功返回0，出错返回非0

可以使用setbuf函数打开或关闭缓冲机制：

-  为了带缓冲进行I/O，参数buf必须指向一个长度为BUFSIZ的缓冲区（该常量定义在<stdio.h>中）。通常在此之后该流就是全缓冲的。
-  为了关闭缓冲，可将buf设置为NULL。

使用setvbuf，我们可以通过指定\ ``mode``\ 参数来精确地说明所需的缓冲类型：

-  \_IOFBF 全缓冲
-  \_IOLBF 行缓冲
-  \_IONBF 不带缓冲

如果指定一个不带缓冲的流，则忽略buf和size参数。

如果指定全缓冲或行缓冲，则buf和size可选择地指定一个缓冲区及其长度。

如果该流是带缓冲的，而buf是NULL，则标准I/O库将自动地为该流分配适当长度的缓冲区。适当长度指的是由常量BUFSIZ所指定的值。

强制冲洗一个流：

.. code:: c

   #include <stdio.h>

   int fflush(FILE* fp);
   // 成功返回0，失败返回EOF

此函数导致该流所有未写的数据都被传送到内核。

打开流
~~~~~~

可以使用以下函数打开一个标准I/O流：

.. code:: c

   #include <stdio.h>

   FILE* fopen(const char* restric pathname, const char* restrict type);
   FILE* freopen(const char* restrict pathname, const char* restrict type, FILE* restrict fp);
   FILE* fdopen(int fd, const char* type);
   // 成功返回文件指针，出错返回NULL

-  fopen函数打开路径名为pathname的一个指定的文件。

-  freopen函数在一个指定的流上打开一个指定的文件，如若该流已经打开，则先关闭该流。若该流已经定向，则使用freopen\ **清除该定向**\ 。

      此函数一般用于将一个指定的文件打开为一个预定义的流：标准输入、标准输出或标准错误。

-  fdopen函数取一个已有的文件描述符，并使一个标准的I/O流与该描述符相结合。

      此函数常用于由创建管道和网络通信通道函数返回的描述符。

type参数指定对该I/O流的读、写方式：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428002826965.png
   :alt: image-20210428002826965

   image-20210428002826965

当以读和写类型打开一个文件时（type中+号），具有下列限制：

-  如果中间没有fflush、fseek、fsetpos或rewind，则在输出的后面不能直接跟随输入。
-  如果中间没有fseek、fsetpos或rewind，或者一个输入操作没有到达文件尾端，则在输入操作之后不能直接跟随输出。

可以调用fclose关闭一个打开的流：

.. code:: c

   #include <stdio.h>

   int fclose(FILE* fp);
   // 成功返回0，出错返回EOF

在文件被关闭之前，冲洗缓冲中的输出数据。缓冲区中的任何输入数据被丢弃。如果标准I/O库已经为该流自动分配了一个缓冲区，则释放此缓冲区。

当一个进程正常终止时（直接调用exit函数，或从main函数返回），则所有带未写缓冲数据的标准I/O流都被冲洗，所有打开的标准I/O流都被关闭。

读和写流
~~~~~~~~

可以使用getc函数来一次读一个字符：

.. code:: c

   #include <stdio.h>

   int getc(FILE* fp);
   int fgetc(FILE* fp);
   int getchar(void);
   // 成功返回下一个字符，若已到达文件尾端或者出错，返回EOF

函数getchar等同于getc(stdin)。

getc与fgetc的区别是：getc可被实现为宏，而fgetc不能实现为宏。

可以使用以下两个函数来判断是否出错和到达文件尾端：

.. code:: c

   #include <stdio.h>

   int ferror(FILE* fp);
   int feof(FILE* fp);
   // 条件为真返回非0，否则返回0

在大多数实现中，为每个流在FILE对象中维护了两个标志：

-  出错标志；
-  文件结束标志。

可以使用clearerr函数清除这两个标志：

.. code:: c

   #include <stdio.h>

   void clearerr(FILE* fp);

可以使用\ ``ungetc()``\ 将字符压送回流中：

.. code:: c

   #include <stdio.h>

   int ungetc(int c, FILE* fp);
   // 成功返回c, 出错返回EOF

压送回到流中的字符以后又可从流中读出，但读出字符的顺序与压送回的顺序相反。

一次成功的ungetc调用会清除该流的文件结束标志，所以已经到达文件尾端时，仍可以回送一个字符。下次读将返回该字符，再读则返回EOF。

   用ungetc压送回字符时，并没有将它们写到底层文件中或设备上，只是将它们写回标准I/O库的流缓冲区中。

可以使用putc函数来一次输出一个字符：

.. code:: c

   #include <stdio.h>

   int putc(int c, FILE* fp);
   int fputc(int c, FILE* fp);
   int putchar(int c);
   // 成功返回c，出错返回EOF

putchar(c)等同于putc(c, stdout)，putc可被实现为宏，而fputc不能实现为宏。

每次一行I/O
~~~~~~~~~~~

可以使用fgets函数来提供每次输入一行的功能：

.. code:: c

   #include <stdio.h>

   char* fgets(char* restrict buf, int n, FILE* restrict fp);
   char* gets(char* buf);  // 不推荐使用，可能造成缓冲区溢出
   // 成功返回buf，若已到达文件尾或出错，则返回NULL

gets删除换行符，fgets保留换行符。

可以使用fputs函数来提供每次输出一行的功能：

.. code:: c

   #include <stdio.h>

   int fputs(const char* restrict str, FILE* restrict fp);
   int puts(const char* str);
   // 成功返回非负值，出错返回EOF

-  fputs将一个以null字节终止的字符串写到指定的流，尾端的终止符null不写出。
-  puts将一个以null字节终止的字符串写到标准输出，终止符不写出。puts随后又将一个换行符写到标准输出。

二进制I/O
~~~~~~~~~

可以使用fread和fwrite来执行二进制I/O操作：

.. code:: c

   #include <stdio.h>

   size_t fread(void* restrict ptr, size_t size, size_t n, FILE* restrict fp);
   size_t fwrite(const void* restrict ptr, size_t size, size_t n, FILE* restrict fp);
   // 返回读或写的对象数

定位流
~~~~~~

有3种方法定位标准I/O流：

1. ftell和fseek函数。文件的位置存放在一个\ **长整型**\ 中。
2. ftello和fseeko函数。文件偏移量使用\ **off_t数据类型**\ 代替了长整型。
3. fgetpos和fsetpos函数。使用\ **抽象数据类型fpos_t**\ 记录文件的位置。这种数据类型可以根据需要定义为一个足够大的数，用以记录文件位置。

..

   需要移植到非UNIX系统上运行的应用程序应当使用fgetpos和fsetpos。

.. code:: c

   #include <stdio.h>

   long ftell(FILE* fp);
   // 成功返回当前文件位置只是，出错返回-1L
   int fseek(FILE* fp, long offset, int whence);
   // 成功返回0，出错返回-1
   void rewind(FILE* fp);

为了定位一个文本文件，whence一定要是SEEK_SET，而且offset只能有两种值：0（后退到文件的起始位置），或是对该文件的ftell所返回的值。

rewind函数将一个流设置到文件的起始位置。

.. code:: c

   #include <stdio.h>

   off_t ftello(FILE* fp);
   // 成功返回当前文件位置，出错返回(off_t)-1
   off_t fseeko(FILE* fp, off_t offset, int whence);
   // 成功返回0，出错返回-1

除了偏移量的类型是off_t而非long以外，ftello函数与ftell相同，fseeko函数与fseek相同。

.. code:: c

   #include <stdio.h>

   int fgetpos(FILE* restrict fp, fpos_t* restrict pos);
   int fsetpos(FILE* fp, const fpos_t* pos);
   // 成功返回0，出错返回非0

fgetpos将文件位置指示器的当前值存入由pos指向的对象中。在以后调用fsetpos时，可以使用此值将流重新定位至该位置。

格式化I/O
~~~~~~~~~

可以使用printf函数来进行格式化输出：

.. code:: c

   #include <stdio.h>

   int printf(const char* restrict format, ...);
   int fprintf(FILE* restrict fp, const char* restrict format, ...);
   int dprintf(int fd, const char* restrict format, ...);
   // 以上三个函数成功返回输出字符数，出错返回负值
   int sprintf(char* restrict buf, const char* restrict format, ...);
   // 成功返回存入数组的字符数，出错返回负值
   int snprintf(char* restrict buf, size_t n, const char* restrict format, ...);
   // 若缓冲区足够大，返回将要存入数组的字符数，出错返回负值

-  printf将格式化数据写到标准输出。

-  fprintf写至指定的流。

-  dprintf写至指定的文件描述符。

-  sprintf将格式化的字符送入数组buf中。

      sprintf在该数组的尾端自动加一个null字节，但该字符不包括在返回值中。

格式说明符不再详述，详细内容请参考APUE第三版P128-P129。

printf族的变体：

.. code:: c

   #include <stdarg.h>
   #include <stdio.h>

   int vprintf(const char* restrict format, va_list arg);
   int vfprintf(FILE* restrict fp, const char* restrict format, va_list arg);
   int vdprintf(int fd, const char* restrict format, va_list arg);
   // 以上三个函数成功返回输出字符数，出错返回负值
   int vsprintf(char* restrict buf, const char* restrict format, va_list arg);
   // 成功返回存入数组的字符数，出错返回负值
   int vsnprintf(char* restrict buf, size_t n, const char* restrict format, va_list arg);
   // 若缓冲区足够大，返回将要存入数组的字符数，出错返回负值

可以使用scanf函数进行格式化输入：

.. code:: c

   #include <stdio.h>

   int scanf(const char* restrict format, ...);
   int fscanf(FILE* restrict fp, const char* restrict format, ...);
   int sscanf(const char* restrict buf, const char* restrict format, ...);
   // 返回赋值的输入项数，若输入出错或在任一转换前已经到达文件尾端，则返回EOF

格式说明符不再详述，详细内容请参考APUE第三版P130。

scanf族的变体：

.. code:: c

   #include <stdarg.h>
   #include <stdio.h>

   int vscanf(const char* restrict format, va_list arg);
   int vfscanf(FILE* restrict fp, const char* restrict format, va_list arg);
   int vsscanf(const char* restrict buf, const char* restrict format, va_list arg);
   // 返回赋值的输入项数，若输入出错或在任一转换前已经到达文件尾端，则返回EOF

实现细节
~~~~~~~~

我们可以对一个流使用fileno函数来获取它的描述符：

.. code:: c

   #include <stdio.h>

   int fileno(FILE* fp);
   // 返回与流相关联的文件描述符

临时文件
~~~~~~~~

可以使用以下两个函数来帮助创建临时文件：

.. code:: c

   #include <stdio.h>

   char* tmpnam(char* ptr);
   // 返回指向唯一路径名的指针
   FILE* tmpfile(void);
   // 成功返回文件指针，出错返回NULL

tmpnam函数产生一个与现有文件名不同的一个有效路径名字符串。每次调用它时，都产生一个不同的路径名，最多调用次数是TMP_MAX(定义在<stdio.h>中)。

-  若ptr是NULL，则所产生的路径名存放在一个静态区中，指向该静态区的指针作为函数值返回。

-  若ptr不是NULL，则它应该是指向长度至少是L_tmpnam个字符的数组（常量L_tmpnam定义在头文件<stdio.h>中）。

使用\ ``tmpnam``\ 和\ ``tmpfile``\ （备注：书上\ ``tmpfile``\ 的地方写的是\ ``tempnam``\ ，估计是写错了，这本书好多小错误啊，翻译和审核都不太认真）函数的缺点：在返回唯一的路径名和用该名字创建文件之间存在一个时间窗口，在这个时间窗口中，另一进程可以用相同的名字创建文件。

我们可以使用以下两个函数来解决这个问题：

.. code:: c

   #include <stdlib.h>

   char* mkdtemp(char* template);
   // 成功返回指向目录名的指针，出错返回NULL
   int mkstemp(char* template);
   // 成功返回文件描述符，出错返回-1

-  mkdtemp函数创建了一个目录，该目录有一个唯一的名字；
-  mkstemp函数创建了一个文件，该文件有一个唯一的名字。

名字是通过template字符串进行选择的。这个字符串是后6位设置为XXXXXX的路径名。函数将这些占位符替换成不同的字符来构建一个唯一的路径名。如果成功的话，这两个函数将\ **修改template字符串**\ 反映临时文件的名字。

使用示例：

.. code:: c

   #include "apue.h"
   #include <errno.h>

   void make_temp(char* template);

   int main() {
       char good_template[] = "/tmp/dirXXXXXX";
       char *bad_template = "/tmp/dirXXXXXX";
       printf("trying to create first temp file...\n");
       make_temp(good_template);
       printf("trying to create second temp file...\n");
       make_temp(bad_template);
       exit(0);
   }

   void make_temp(char* template) {
       int fd;
       struct stat sbuf;
       if ((fd = mkstemp(template)) < 0) {
           err_sys("can't create temporary file");
       }
       printf("temp name = %s\n", template);
       close(fd);
       if (stat(template, &sbuf) < 0) {
           if (errno == ENOENT) {
               printf("file doesn't exist\n");
           } else {
               err_sys("stat failed");
           }
       } else {
           printf("file exists\n");
           unlink(template);
       }
   }

输出结果：

::

   trying to create first temp file...
   temp name = /tmp/dirKOBzQc
   file exists
   trying to create second temp file...
   Segmentation fault (core dumped)

内存流
~~~~~~

我们可以使用fmemopen函数进行内存流的创建：

.. code:: c

   #include <stdio.h>

   FILE* fmemopen(void *restrict buf, size_t size, const char* restrict type);
   // 成功返回流指针，失败返回NULL

type参数控制如何使用流：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428013218628.png
   :alt: image-20210428013218628

   image-20210428013218628

-  无论何时以追加写方式打开内存流时，当前文件位置设为缓冲区中的第一个null字节。如果缓冲区中不存在null字节，则当前位置就设为缓冲区结尾的后一个字节。当流并不是以追加写方式打开时，当前位置设为缓冲区的开始位置。
-  如果buf参数是一个null指针，打开流进行读或者写都没有任何意义。因为在这种情况下缓冲区是通过fmemopen进行分配的，没有办法找到缓冲区的地址，只写方式打开流意味着无法读取已写入的数据，同样，以读方式打开流意味着只能读取那些我们无法写入的缓冲区中的数据。
-  任何时候需要增加流缓冲区中数据量以及调用fclose、fflush、fseek、fseeko以及fsetpos时都会在当前位置写入一个null字节。

使用示例：

.. code:: c

   #include "apue.h"

   #define BSZ 48

   int main() {
       FILE* fp;
       char buf[BSZ];
       memset(buf, 'a', BSZ-2);
       buf[BSZ-2] = '\0';
       buf[BSZ-1] = 'X';
       if ((fp = fmemopen(buf, BSZ, "w+")) == NULL) {
           err_sys("fmemopen failed");
       }
       printf("initial buffer contents: %s\n", buf);
       fprintf(fp, "hello, world");
       printf("before flush: %s\n", buf);
       fflush(fp);
       printf("after flush: %s\n", buf);
       printf("len of string in buf = %ld\n", (long)strlen(buf));

       memset(buf, 'b', BSZ-2);
       buf[BSZ-2] = '\0';
       buf[BSZ-1] = 'X';
       fprintf(fp, "hello, world!");
       fseek(fp, 0, SEEK_SET);
       printf("after fseek: %s\n", buf);
       printf("len of string in buf = %ld\n", (long)strlen(buf));

       memset(buf, 'c', BSZ-2);
       buf[BSZ-2] = '\0';
       buf[BSZ-1] = 'X';
       fprintf(fp, "hello, world");
       fclose(fp);
       printf("after fclose: %s\n", buf);
       printf("len of string in buf = %ld\n", (long)strlen(buf));
       return 0;
   }

输出结果：

::

   initial buffer contents: 
   before flush: 
   after flush: hello, world
   len of string in buf = 12
   after fseek: bbbbbbbbbbbbhello, world!
   len of string in buf = 25
   after fclose: hello, worldcccccccccccccccccccccccccccccccccc
   len of string in buf = 46

还可以使用open_memstream和open_wmemstream函数来创建内存流：

.. code:: c

   #include <stdio.h>

   FILE* open_memstream(char** bufp, size_t *sizep);

   #include <wchar.h>

   FILE* openwmemstream(wchar_t** bufp, size_t *sizep);
   // 以上两个函数，成功时返回流指针，出错时返回NULL

open_memstream函数创建的流是面向字节的，open_wmemstream函数创建的流是面向宽字节的。

这两个函数与fmemopen函数的不同在于：

-  创建的流只能写打开；
-  不能指定自己的缓冲区，但可以分别通过bufp和sizep参数访问缓冲区地址和大小；
-  关闭流后需要自行释放缓冲区；
-  对流添加字节会增加缓冲区大小。

在缓冲区地址和大小的使用上必须遵循一些原则：

1. 缓冲区地址和长度只有在调用fclose或fflush后才有效；
2. 这些值只有在下一次流写入或调用fclose前才有效。
