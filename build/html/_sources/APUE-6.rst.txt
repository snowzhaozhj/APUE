系统数据文件和信息
------------------

口令文件
~~~~~~~~

口令文件是\ ``/etc/passwd``\ ，是一个ASCII文件。

为了阻止一个特定用户登录系统：

-  可以将登录shell设置为/dev/null外
-  还可以将登录shell设置为/dev/false。它简单地以不成功（非0）状态终止，该shell将此种终止状态判断为假。
-  还可以将登录shell设置为/bin/true。它所做的一切是以成功（0）状态终止。
-  某些系统提供nologin命令，它打印可定制的出错信息，然后以非0状态终止。

使用nobody用户名的一个目的是，使任何人都可登录至系统，

某些系统提供了\ ``vipw``\ 来编辑口令文件(需要管理员权限)。

可以通过以下两个函数来获取口令文件项：

.. code:: c

   #include <pwd.h>

   struct passwd* getpwuid(uid_t uid);
   struct passwd* getpwnam(const char* name);
   // 成功返回指针，出错返回NULL

其中\ ``struct passwd``\ 的结构如下图所示：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428123359734.png
   :alt: image-20210428123359734

   image-20210428123359734

可以使用以下函数来查看整个口令文件：

.. code:: c

   #include <pwd.h>

   struct passwd* getpwent(void);
   // 成功返回指针，出错或者到达文件尾端，则返回NULL
   void setpwent(void);
   void endpwent(void);

-  getpwent函数返回口令文件中的下一个记录项。
-  setpwent将getpwent的读写地址指向密码文件开头。
-  endpwent关闭这些文件。

getpwnam函数的一种实现：

.. code:: c

   struct passwd* getpwnam(const char* name) {
       struct passwd *ptr;
       setpwent();
       while ((ptr = getpwent()) != NULL) {
           if (strcmp(name, ptr->pw_name) == 0) {
               break;
           }
       }
       endpwent();
       return ptr;
   }

阴影口令
~~~~~~~~

加密口令是经单向加密算法处理过的用户口令副本。此算法是单向的，不能从加密口令猜测到原来的口令。

可以使用以下函数访问阴影口令文件：

.. code:: c

   #include <shadow.h>

   struct spwd* getspwnam(const char* name);
   struct spwd* getspent(void);
   // 成功返回指针，出错返回NULL
   void setspent(void);
   void endspent(void);

其中\ ``struct spwd``\ 的结构如下图所示：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428123720350.png
   :alt: image-20210428123720350

   image-20210428123720350

组文件
~~~~~~

可以使用以下两个函数来查看组名或数值组ID：

.. code:: c

   #include <grp.h>

   struct group* getgrgid(gid_t gid);
   struct group* getgrnam(const char* name);
   // 成功返回指针，出错或者到达文件尾端，则返回NULL

其中\ ``struct group``\ 的结构如下图所示：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428124014848.png
   :alt: image-20210428124014848

   image-20210428124014848

可以使用以下几个函数来搜索整个组文件：

.. code:: c

   #include <grp.h>

   struct group* getgrent(void);
   // 成功返回指针，出错或者到达文件尾端，返回NULL
   void setgrent(void);
   void endgrent(void);

附属组ID
~~~~~~~~

可以通过以下函数来获取和设置附属组ID：

.. code:: c

   #include <unistd.h>

   int getgroups(int gidsetsize, git_t grouplist[]);
   // 成功返回附属组ID数量，出错返回-1

   #include <grp.h>    // in linux

   int setgroups(int ngroups, const git_t grouplist[]);
   int initgroups(const char* username, gid_t basegid);
   // 成功返回0，出错返回-1

-  getgroups将进程所属用户的各附属组ID填写到数组grouplist中，填写入该数组的附属组ID数最多为gidsetsize个。实际填写到数组中的附属组ID数由函数返回。若gidsetsize为0，则函数只返回附属组ID数，而对数组grouplist则不做修改。

-  setgroups可由超级用户调用以便为调用进程设置附属组ID表。grouplist是组ID数组，而ngroups说明了数组中的元素数。ngroups的值不能大于NGROUPS_MAX。

-  initgroups读整个组文件（用数getgrent、setgrent和endgrent），然后对username确定其组的成员关系。然后，它调用setgroups，以便为该用户初始化附属组ID表。除了在组文件中找到
   username所在的所有组，initgroups也在附属组ID表中包括了basegid。basegid是username在口令文件中的组ID。

其他数据文件
~~~~~~~~~~~~

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428133409728.png
   :alt: image-20210428133409728

   image-20210428133409728

登录账户记录
~~~~~~~~~~~~

记录所用的结构体：

.. code:: c

   struct utmp {
       char ut_line[8];    // tty line: "ttyh0", "ttyp0", ...
       char ut_name[8];    // login name
       long ut_time;       // seconds since Epoch
   };

登录时，login程序填写此类型结构，将其写入到utmp文件和wtmp文件中。

注销时，init进程将utmp文件中相应的记录擦除（每个字节都填以null字节），并将一个新记录添写到wtmp文件中。

在wtmp文件的注销记录中，ut_name字段清除为0。

在系统再启动时，以及更改系统时间和日期的前后，都在wtmp文件中追加写特殊的记录项。

who(1)程序读取utmp文件，并以可读格式打印其内容。

last(1)命令，它读wtmp文件并打印所选择的记录。

系统标识
~~~~~~~~

可以使用uname函数来查看与操作系统相关信息：

.. code:: c

   #include <sys/utsname.h>

   int uname(struct utsname* name);
   // 成功返回非负值，出错返回-1

``struct utsname``\ 的结构：

.. code:: c

   struct utsname {
       char sysname[];     // name of the OS
       char nodename[];    // name of this node
       char release[];     // current release of OS
       char version[];     // current version of this release
       char machine[];     // name of hardware type
   };

可以使用uname(1)来打印utsname中的信息。

可以使用gethostname函数来查看主机名：

.. code:: c

   #include <unistd.h>

   int gethostname(char* name, int namelen);
   // 成功返回0，出错返回-1

可以通过hostname(1)命令来获取和设置主机名。

时间和日期例程
~~~~~~~~~~~~~~

time函数返回当前时间和日期。

.. code:: c

   #include <time.h>

   time_t time(time_t *calptr);
   // 成功返回时间值，出错返回-1

还可以通过clock_gettime函数来获取指定时钟的时间：

.. code:: c

   #include <sys/time.h>

   int clock_gettime(clockid_t clock_id, struct timespec *tsp);
   // 成功返回0，出错返回-1

其中\ ``clock_id``\ 的标准值：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210428134858155.png
   :alt: image-20210428134858155

   image-20210428134858155

当时钟ID设置为CLOCK_REALTIME时，clock_gettime函数提供了与time函数类似的功能，不过在系统支持高精度时间值的情况下，clock_gettime可能比time函数得到更高精度的时间值。

可以使用clock_getres函数来调整时钟精度：

.. code:: c

   #include <sys/time.h>

   int clock_getres(clockid_t clock_id, struct timespec *tsp);
   // 成功返回0，出错返回-1

clock_getres函数把参数tsp指向的timespec结构初始化为与clock_id参数对应的时钟精度。例如，如果精度为1毫秒，则tv_sec字段就是0，tv_nsec字段就是1
000 000。

我们可以使用clock_settime来给特定的时钟设定时间：

.. code:: c

   #include <sys/time.h>

   int clock_settime(clockid_t clock_id, const struct timespec *tsp);
   // 成功返回0，出错返回-1

SUSv4指定gettimeofday函数现在已弃用。然而，一些程序仍然使用这个函数，因为与time函数相比，gettimeofday提供了更高的精度（可到微秒级）。

.. code:: c

   #include <sys/time.h>

   int gettimeofday(struct timeval* restrict tp, void* restrict tzp); // tzp的唯一合法值是NULL
   // 总是返回0

可以使用localtime和gmtime将日历时间转换成分解的时间：

.. code:: c

   #include <time.h>

   struct tm *gmtime(const time_t *calptr);
   struct tm *localtime(const time_t *calptr);
   // 成功返沪指向分解的tm结构的指针，出错返回NULL

其中\ ``struct tm``\ 的结构为：

.. code:: c

   struct tm {         /* a broken-down time */
       int tm_sec;     /* seconds after the minute: [0 - 60] */
       int tm_min;     /* minutes after the hour: [0 - 59] */
       int tm_hour;    /* hours after midnight: [0 - 23] */
       int tm_mday;    /* day of the month: [1 - 31] */
       int tm_mon;     /* months since January: [0 - 11] */
       int tm_year;    /* years since 1900 */
       int tm_wday;    /* days since Sunday: [0 - 6] */
       int tm_yday;    /* days since January 1: [0 - 365] */
       int tm_isdst;   /* daylight saving time flag: <0, 0, >0 */
   };

可以通过mktime函数，以本地时间的年、月、日等作为参数，将其变换成time_t值。

.. code:: c

   #include <time.h>

   time_t mktime(struct tm *tmptr);
   // 返回值：若成功，返回日历时间；若出错，返回-1

可以使用strftime函数来格式化输出时间：

.. code:: c

   #include <time.h>

   size_t strftime(char *restrict buf, size_t maxsize, const char *restrict format, const struct tm *restrict tmptr);
   size_t strftime_l(char *restrict buf, size_t maxsize, const char *restrict format, const struct tm *restrict tmptr, locale_t locale);
   // 若buf空间足够大则返回存入数组的字符数，否则返回0

strftime_l允许调用者将区域指定为参数，除此之外，strftime和strftime_l函数是相同的。

tmptr参数是要格式化的时间值，由一个指向分解时间值tm结构的指针说明。格式化结果存放在一个长度为maxsize个字符的buf数组中，如果buf长度足以存放格式化结果及一个null终止符，则该函数返回在buf中存放的字符数（不包括null终止符）；否则该函数返回0。

format参数控制时间值的格式。

一个使用示例：

.. code:: c

   #include <stdio.h>
   #include <stdlib.h>
   #include <time.h>

   int main() {
       time_t t;
       struct tm* tmp;
       char buf1[16];
       char buf2[64];

       time(&t);
       tmp = localtime(&t);
       if (strftime(buf1, 16, "time and date: %r, %a %b %d, %Y", tmp) == 0) {
           printf("buffer length 16 is too small\n");
       } else {
           printf("%s\n", buf1);
       }
       if (strftime(buf2, 64, "time and date: %r, %a %b %d, %Y", tmp) == 0) {
           printf("buffer length 64 is too small\n");
       } else {
           printf("%s\n", buf2);
       }
       exit(0);
   }

输出结果：

::

   buffer length 16 is too small
   time and date: 02:28:54 PM, Wed Apr 28, 2021

strptime函数是strftime的逆向版本，把字符串时间转换成分解时间。

.. code:: c

   #include <time.h>

   char *strptime(const char *restrict buf, const char *restrict format, struct tm *restrict tmptr);
   // 返回值：指向上次解析的字符的下一个字符的指针；否则，返回NULL

格式说明符：
