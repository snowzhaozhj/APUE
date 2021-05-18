网络IPC：套接字
---------------

套接字描述符
~~~~~~~~~~~~

套接字描述符在UNIX系统中被当作是一种文件描述符。许多处理文件描述符的函数（如read和write）可以用于处理套接字描述符。

可以用socket函数来创建一个套接字：

.. code:: c

   #include <sys/socket.h>

   int socket(int domain, int type, int protocol);
   // 成功返回套接字描述符，出错返回-1

其中domain的可选值有：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512182404406.png
   :alt: image-20210512182404406

   image-20210512182404406

AF表示地址族(address family)

参数type确定套接字的类型，有如下可选值：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512182534400.png
   :alt: image-20210512182534400

   image-20210512182534400

参数protocol的可能值有：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512182648987.png
   :alt: image-20210512182648987

   image-20210512182648987

参数protocol通常是0，表示为给定的域和套接字类型选择默认协议。当对同一域和套接字类型支持多个协议时，可以使用protocol选择一个特定协议。

数据报(SOCK_DGRAM)提供无连接服务。

字节流(SOCK_STREAM)提供字节流服务(有连接的)，应用程序分辨不出报文的界限。从SOCK_STREAM套接字读数据时，它也许不会返回所有由发送进程所写的字节数。要获得发送过来的所有数据，可能需要经过多次函数调用。

SOCK_DGRAM的默认协议是UDP，SOCK_STREAM的默认协议是TCP。

SOCK_SEQPACKET套接字和SOCK_STREAM套接字很类似，只是从该套接字得到的是基于报文的服务而不是字节流服务。也就是说从SOCK_SEQPACKET套接字接收的数据量与对方所发送的一致。

SOCK_RAW套接字提供一个数据报接口，用于直接访问下面的网络层（即因特网域中的
IP层）。使用这个接口时，应用程序负责构造自己的协议头部，因为传输协议（如TCP和UDP）被绕过了。

   当创建一个原始套接字时，需要有超级用户特权。

常见I/O函数对套接字描述符的支持：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512183520951.png
   :alt: image-20210512183520951

   image-20210512183520951

套接字通信是双向的。可以采用shutdown函数来禁止一个套接字的I/O。

.. code:: c

   #include <sys/socket.h>

   int shutdown(int sockfd, int how);
   // 成功返回0，出错返回−1

-  如果how是SHUT_RD（关闭读端），那么无法从套接字读取数据。
-  如果how是SHUT_WR（关闭写端），那么无法使用套接字发送数据。
-  如果how是SHUT_RDWR，则既无法读取数据，又无法发送数据。

寻址
~~~~

字节序
^^^^^^

大端字节序：最高有效字节的字节地址最低。

小端字节序：最低有效字节的字节地址最低。

TCP/IP协议栈使用大端字节序。

4个用来在处理器字节序和网络字节序中转换的函数：

.. code:: c

   #include <arpa/inet.h>

   uint32_t htonl(uint32_t hostint32);
   // 返回以网络字节序表示的32位整数
   uint16_t htons(uint16_t hostint16);
   // 返回以网络字节序表示的16位整数
   uint32_t ntohl(uint32_t netint32);
   // 返回以主机字节序表示的32位整数
   uint16_t ntohs(uint16_t netint16);
   // 返回以主机字节序表示的16位整数

h代表主机host，n代表网络network，l代表长long，s代表短short。

地址格式
^^^^^^^^

一个地址标识一个特定通信域的套接字端点。

为使不同格式地址能够传入到套接字函数，地址会被强制转换成一个\ **通用的地址结构**\ sockaddr：

.. code:: c

   struct sockaddr {
       sa_family_t sa_family;  // address family: AF_INET, AF_INET6, ...
       char sa_data[];         // variable-length address
   };

套接字的实现可以自由地添加额外的成员并且定义sa_data成员的大小。

因特网地址定义在<netinet/in.h>头文件中。IPv4中套接字地址用结构sockaddr_in表示：

.. code:: c

   struct in_addr {
       in_addr_t s_addr;   // IPv4 address
   };

   struct sockaddr_in {
       sa_family_t sin_family; // address family
       in_port_t sin_port;     // port number
       struct in_addr sin_addr;// IPv4 address
   };

而IPv6中使用结构sockaddr_in6表示：

.. code:: c

   struct in6_addr {
       uint8_t s6_addr[16];    // IPv6 address
   };

   struct sockaddr_in6 {
       sa_family_t sin6_family;    // address family
       in_port_t sin6_port;        // port number
       uint32_t sin_flowinfo;      // traffic class and flow info
       struct in6_addr sin6_addr;  // IPv6 address
       uint32_t sin6_scope_id;     // set of interfaces for scope
   };

这些都是SUS的定义，具体实现可以自由添加更多的字段。

**尽管sockaddr_in与sockaddr_in6结构相差比较大，但它们均被强制转换成sockaddr结构输入到套接字例程中。**

套接字二进制地址格式和点分十进制字符表示之间的相互转换：

.. code:: c

   #include <arpa/inet.h>

   const char *inet_ntop(int domain, const void *restrict addr, char *restrict str, socklen_t size);
   // 成功返回地址字符串指针，出错返回NULL
   int inet_pton(int domain, const char *restrict str, void *restrict addr);
   // 成功返回1，格式无效返回0，出错返回-1

-  inet_ntop将网络字节序的二进制地址转换成文本字符串格式。
-  inet_pton将文本字符串格式转换成网络字节序的二进制地址。

参数domain仅支持两个值：AF_INET和AF_INET6。

参数size通常取INET_ADDRSTRLEN来存放一个表示IPv4地址的文本字符串；取INET6_ADDRSTRLEN来存放一个表示IPv6地址的文本字符串。

地址查询
^^^^^^^^

可以调用gethostent函数来查找给定计算机系统的主机信息：

.. code:: c

   #include <netdb.h>

   struct hostent *gethostent(void);
   // 成功返回指针，出错返回NULL
   void sethostent(int stayopen);
   void endhostent(void);

如果主机数据库文件没有打开，gethostent会打开它。函数gethostent返回文件中的下一个条目。

函数sethostent会打开文件，如果文件已经被打开，那么将其回绕。当stayopen参数设置成非0值时，调用gethostent之后，文件将依然是打开的。

   不太明白回绕是什么意思。

函数endhostent可以关闭文件。

当gethostent返回时，会得到一个指向hostent结构的指针，该结构可能包含一个静态的数据缓冲区，\ **每次调用gethostent，缓冲区都会被覆盖**\ 。

其中struct hostent结构至少包含以下成员：

.. code:: c

   struct hostent {
       char *h_name;       // name of host
       char **h_aliases;   // pointer to alternate host name array
       int h_addrtype;     // address type
       int h_length;       // length in bytes of address
       char **h_addr_list; // pointer to array of network addresses
   }

返回的地址按照网络字节序。地址类型(h_addrtype)是AF_INET系列常量。

可以使用以下函数来获取网络名和网络编号：

.. code:: c

   #include <netdb.h>

   struct netent *getnetbyaddr(uint32_t net, int type);
   struct netent *getnetbyname(const char *name);
   struct netent *getnetent(void);
   // 成功返回指针，出错返回NULL
   void setnetent(int stayopen);
   void endnetent(void);

其中netent的结构至少包含以下字段：

.. code:: c

   struct netent {
       char *n_name;       // network name
       char **n_aliases;   // alternate network name array pointer
       int n_addrtype;     // address type
       uint32_t n_net;     // network number
   };

网络编号按照网络字节序返回。地址类型是地址族常量之一（如AF_INET）。

可以使用以下函数在协议名字和协议编号之间进行映射：

.. code:: c

   #include <netdb.h>

   struct protoent *getprotobyname(const char *name);
   struct protoent *getprotobynumber(int proto);
   struct protoent *getprotoent(void);
   // 成功返回指针，出错返回NULL
   void setprotoent(int stayopen);
   void endprotoent(void);

其中protoent结构至少包含以下成员：

.. code:: c

   struct protoent {
       char *p_name;       // protocol name
       char **p_aliases;   // pointer to alternate protocol name array
       int p_proto;        // protocol number
   };

服务是由地址的端口号部分表示的。每个服务由一个唯一的众所周知的端口号来支持。可以使用函数getservbyname将一个服务名映射到一个端口号，使用函数getservbyport将一个端口号映射到一个服务名，使用函数getservent顺序扫描服务数据库。

.. code:: c

   #include <netdb.h>

   struct servent *getservbyname(const char *name, const char *proto);
   struct servent *getserbyport(int port, const char *proto);
   struct servent *getservent(void);
   // 成功返回指针，出错返回NULL
   void setservent(int stayopen);
   void endservent(void);

其中servent结构至少包含以下成员：

.. code:: c

   struct servent{
       char *s_name;       /* service name */
       char **s_aliases;   /* pointer to alternate service name array */
       int s_port;         /* port　number */
       char *s_proto;      /* name of protocol */
   };

可以使用getaddrinfo函数将一个主机名和一个服务名映射到一个地址：

.. code:: c

   #include <sys/socket.h>
   #include <netdb.h>

   int getaddrinfo(const char *restrict host, const char *restrict sevice, const struct addrinfo *restrict hint, struct addrinfo **restrict res);
   // 成功返回0，出错返回非0错误码
   void freeaddrinfo(struct addrinfo *ai);

getaddrinfo函数返回一个链表结构addrinfo。

其中addrinfo结构至少包含以下成员：

.. code:: c

   struct addrinfo {
       int ai_flags;               // customize behavior
       int ai_family;              // address family
       int ai_socktype;            // socket type
       int ai_protocol;            // protocol
       socklen_t ai_addrlen;       // length in bytes of address
       struct sockaddr *ai_addr;   // address
       char *ai_cannoname;         // canonical name of host
       struct addrinfo *ai_next;   // next in list
   };

freeaddrinfo可以释放一个或多个这种结构，这取决于用ai_next字段链接起来的结构有多少。

hint是一个用于过滤地址的模板，包括ai_family、ai_flags、ai_protocol和ai_socktype字段。剩余的整数字段必须设置为0，指针字段必须为空。

其中flags的可选值有：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512220244767.png
   :alt: image-20210512220244767

   image-20210512220244767

如果getaddrinfo失败，需要使用gai_strerror函数将返回的错误码转换为错误消息：

.. code:: c

   #include <netdb.h>

   const char *gai_strerror(int error);
   // 返回指向描述错误的字符串的指针

getnameinfo函数将一个地址转换成一个主机名和一个服务名：

.. code:: c

   #include <sys/socket.h>
   #include <netdb.h>

   int getnameinfo(const struct sockaddr *restrict addr, socklen_t alen, char *restrict host, socklen_t hostlen, char *restrict service, socklen_t servlen, int flags);
   // 成功返回0，出错返回非0值

套接字地址（addr）被翻译成一个主机名和一个服务名。

如果host非空，则指向一个长度为hostlen字节的缓冲区用于存放返回的主机名。

如果service非空，则指向一个长度为servlen字节的缓冲区用于存放返回的主机名。

其中flags参数的可选值如下：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512220753566.png
   :alt: image-20210512220753566

   image-20210512220753566

一个示例程序：

.. code:: c

   #include "apue.h"
   #if defined(SOLARIS)
   #include <netinet/in.h>
   #endif
   #include <netdb.h>
   #include <arpa/inet.h>
   #if defined(BSD)
   #include <sys/socket.h>
   #include <netinet/in.h>
   #endif

   void print_family(struct addrinfo* aip) {
       printf(" family ");
       switch (aip->ai_family) {
           case AF_INET:
               printf("inet");
               break;
           case AF_INET6:
               printf("inet6");
               break;
           case AF_UNIX:
               printf("unix");
               break;
           case AF_UNSPEC:
               printf("unspecified");
               break;
           default:
               printf("unknown");
       }
   }

   void print_type(struct addrinfo* aip) {
       printf(" type ");
       switch (aip->ai_socktype) {
           case SOCK_STREAM:
               printf("stream");
               break;
           case SOCK_DGRAM:
               printf("datagram");
               break;
           case SOCK_SEQPACKET:
               printf("seqpacket");
               break;
           case SOCK_RAW:
               printf("raw");
               break;
           default:
               printf("unknown (%d)", aip->ai_socktype);
       }
   }

   void print_protocol(struct addrinfo* aip) {
       printf(" protocol ");
       switch (aip->ai_protocol) {
           case 0:
               printf("default");
               break;
           case IPPROTO_TCP:
               printf("TCP");
               break;
           case IPPROTO_UDP:
               printf("UDP");
               break;
           case IPPROTO_RAW:
               printf("raw");
               break;
           default:
               printf("unknow (%d)", aip->ai_protocol);
       }
   }

   void print_flags(struct addrinfo* aip) {
       printf("flags");
       if (aip->ai_flags == 0) {
           printf(" 0");
       } else {
           if (aip->ai_flags & AI_PASSIVE) {
               printf(" passive");
           }
           if (aip->ai_flags & AI_CANONNAME) {
               printf(" canon");
           }
           if (aip->ai_flags & AI_NUMERICHOST) {
               printf(" numhost");
           }
           if (aip->ai_flags & AI_NUMERICSERV) {
               printf(" numserv");
           }
           if (aip->ai_flags & AI_V4MAPPED) {
               printf(" v4mapped");
           }
           if (aip->ai_flags & AI_ALL) {
               printf(" all");
           }
       }
   }

   int main(int argc, char* argv[]) {
       struct addrinfo *ailist, *aip;
       struct addrinfo hint;
       struct sockaddr_in *sinp;
       const char *addr;
       int err;
       char abuf[INET_ADDRSTRLEN];

       if (argc != 3) {
           err_quit("usage: %s <nodename> <service>", argv[0]);
       }
       hint.ai_flags = AI_CANONNAME;
       hint.ai_family = 0;
       hint.ai_socktype = 0;
       hint.ai_protocol = 0;
       hint.ai_addrlen = 0;
       hint.ai_canonname = NULL;
       hint.ai_addr = NULL;
       hint.ai_next = NULL;
       if ((err = getaddrinfo(argv[1], argv[2], &hint, &ailist)) != 0) {
           err_quit("getaddrinfo error: %s", gai_strerror(err));
       }
       for (aip = ailist; aip != NULL; aip = aip->ai_next) {
           print_flags(aip);
           print_family(aip);
           print_protocol(aip);
           printf("\n\thost %s", aip->ai_canonname ? aip->ai_canonname : "-");
           if (aip->ai_family == AF_INET) {
               sinp = (struct sockaddr_in*)aip->ai_addr;
               addr = inet_ntop(AF_INET, &sinp->sin_addr, abuf, INET_ADDRSTRLEN);
               printf(" address %s", addr ? addr : "unknown");
               printf(" port %d", ntohs(sinp->sin_port));
           }
           printf("\n");
       }
       exit(0);
   }

将套接字与地址关联
^^^^^^^^^^^^^^^^^^

使用bind函数来将地址和套接字关联起来：

.. code:: c

   #include <sys/socket.h>

   int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
   // 成功返回0，出错返回-1

..

   如果调用connect或listen时，还没有将地址绑定到套接字上，系统会选一个地址绑定到套接字上。

可以调用getsockname函数来获取绑定到套接字上的地址：

.. code:: c

   #include <sys/socket.h>

   int getsockname(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict alenp);
   // 成功返回0，出错返回-1

调用getsockname之前，alenp指向整数指定缓冲区sockaddr的长度。返回时，该整数会被设置成返回地址的大小。如果地址和提供的缓冲区长度不匹配，地址会被自动截断而不报错。如果当前没有地址绑定到该套接字，则其结果是未定义的。

如果套接字已经和对等方连接，可以调用getpeername函数来找到对方的地址。

.. code:: c

   #include <sys/socket.h>

   int getpeername(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict alenp);
   // 成功返回0，出错返回-1

建立连接
~~~~~~~~

在处理一个面向连接的网络服务（SOCK_STREAM或SOCK_SEQPACKET）的时候，需要在请求服务的进程套接字（客户端）和提供服务的进程套接字（服务器）之间建立一个连接。

使用connect函数来建立连接：

.. code:: c

   #include <sys/socket.h>

   int connect(int sockfd, const struct sockaddr *addr, socklen_t len);
   // 成功返回0，出错返回-1

addr参数指定我们想要与之通信的服务器地址。

如果sockfd此时没有绑定到一个地址，connect会给调用者绑定一个默认地址。

   connect函数还可以用于无连接的网络服务（SOCK_DGRAM）。如果用SOCK_DGRAM套接字调用connect，传送的报文的目标地址会设置成connect调用中所指定的地址，这样每次传送报文时就不需要再提供地址。并且仅能接收来自指定地址的报文。

服务器调用listen函数来宣告它愿意接受连接请求：

.. code:: c

   #include <sys/socket.h>

   int listen(int sockfd, int backlog);
   // 成功返回0，出错返回-1

backlog参数指定了该进程所要入队的未完成连接请求数量。队列满了之后，系统就会拒绝多余的连接请求。

调用了listen后套接字就能接收连接请求。可以使用accept函数获得连接请求并建立连接。

.. code:: c

   #include <sys/socket.h>

   int accept(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict len);
   // 成功返回文件(套接字)描述符，出错返回−1

返回的文件描述符是套接字描述符，该描述符连接到调用connect的客户端。这个新的套接字描述符和原始套接字（sockfd）具有相同的套接字类型和地址族。传给accept的原始套接字没有关联到这个连接，而是继续保持可用状态并接收其他连接请求。

返回时，accept会将addr设置为客户端的地址，并且更新指向len的整数来反映该地址的大小。不关心这两个参数可将两者设置为NULL。

如果没有连接请求在等待，accept会阻塞直到一个请求到来。如果sockfd处于非阻塞模式，accept会返回−1，并将errno设置为EAGAIN或EWOULDBLOCK。

数据传输
~~~~~~~~

有3个用来发送数据的函数。

.. code:: c

   #include <sys/socket.h>

   ssize_t send(int sockfd, const void *buf, size_t nbytes, int flags);
   // 成功返回发送的字节数，出错返回-1

使用send时，套接字必须已经连接。

buf参数和nbytes参数与write函数中相同。

flags参数的可选值如下：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512223610727.png
   :alt: image-20210512223610727

   image-20210512223610727

..

   即使send成功返回，也并不表示连接的另一端的进程就一定接收了数据。我们所能保证的只是当send成功返回时，数据已经被无错误地发送到网络驱动程序上。

   对于支持报文边界的协议，如果尝试发送的单个报文的长度超过协议所支持的最大长度，那么send会失败，并将errno设为EMSGSIZE。对于字节流协议，send会阻塞直到整个数据传输完成。

sendto在send的基础上可以指定一个目标地址(用于无连接的套接字)：

.. code:: c

   #include <sys/socket.h>

   ssize_t sendto(int sockfd, const void *buf, size_t nbytes, int flags, const struct sockaddr *destaddr, socklen_t destlen);
   // 成功返回发送的字节数，出错返回-1

..

   对于面向连接的套接字，目标地址会被忽略。

sendmsg函数类似于writev函数：

.. code:: c

   #include <sys/socket.h>

   ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
   // 成功返回发送的字节数，出错返回-1

其中msghdr的结构如下：

.. code:: c

   struct msghdr {
       void *msg_name;             // optional address
       socklen_t msg_namelen;      // address size in bytes
       struct iovec *msg_iov;      // array of I/O buffers
       int msg_iovlen;             // number of elements in array
       void *msg_control;          // ancillary data, 辅助数据
       socklen_t msg_controllen;   // number of ancillary data
       int msg_flags;  
   };

有三个用来接受数据的函数：

recv函数类似于read函数：

.. code:: c

   #include <sys/socket.h>

   ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags);
   // 返回数据的字节长度，若无可用数据或对等方已经按序结束则返回0，出错返回-1

其中flags参数的可选值有：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512225054807.png
   :alt: image-20210512225054807

   image-20210512225054807

当指定MSG_PEEK标志时，可以查看下一个要读取的数据但不真正取走它。当再次调用read或其中一个recv函数时，会返回刚才查看的数据。

对于SOCK_STREAM套接字，接收的数据可以比预期的少。MSG_WAITALL标志会阻止这种行为，直到所请求的数据全部返回，recv函数才会返回。对于SOCK_DGRAM和SOCK_SEQPACKET套接字，MSG_WAITALL
标志没有改变什么行为，因为这些基于报文的套接字类型一次读取就返回整个报文。

如果发送者已经调用shutdown来结束传输，或者网络协议支持按默认的顺序关闭并且发送端已经关闭，那么当所有的数据接收完毕后，recv返回0。

可以使用recvfrom函数来得到数据发送者的源地址：

.. code:: c

   #include <sys/socket.h>

   ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags, struct sockaddr *restrict addr, socklen_t *restrict addrlen);
   // 返回数据的字节长度，若无可用数据或对等方已经按序结束则返回0，出错返回-1

recvmsg函数类似于readv函数：

.. code:: c

   #include <sys/socket.h>

   ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
   // 返回数据的字节长度，若无可用数据或对等方已经按序结束则返回0，出错返回-1

套接字选项
~~~~~~~~~~

可以使用setsockopt函数来设置套接字选项。

.. code:: c

   #include <sys/socket.h>

   int setsockopt(int sockfd, int option, const void *val, socklen_t len);
   // 成功返回0，出错返回-1

参数level标识了选项应用的协议。如果选项是通用的套接字层次选项，则level设置成SOL_SOCKET。否则，level设置成控制这个选项的协议编号。对于TCP选项，level是IPPROTO_TCP，对于IP，level是IPPROTO_IP。

option参数的可选值如下：

.. figure:: https://gitee.com/snow_zhao/img/raw/master/img/image-20210512230038747.png
   :alt: image-20210512230038747

   image-20210512230038747

参数val根据选项的不同指向一个数据结构或者一个整数。一些选项是on/off开关。如果整数非0，则启用选项。如果整数为0，则禁止选项。参数len指定了val指向的对象的大小。

可以使用getsockopt函数来获取选项的当前值：

.. code:: c

   #include <sys/socket.h>

   int getsockopt(int sockfd, int level, int option, void *restrict val, socklen_t *restric lenp);
   // 成功返回0，出错返回-1

参数lenp是一个指向整数的指针。在调用getsockopt之前，设置该整数为复制选项缓冲区的长度。如果选项的实际长度大于此值，则选项会被截断。如果实际长度正好小于此值，那么返回时将此值更新为实际长度。

带外数据
~~~~~~~~

带外数据（out-of-band
data）是一些通信协议所支持的可选功能，与普通数据相比，它允许更高优先级的数据传输。带外数据先行传输，即使传输队列已经有数据。

TCP支持带外数据，但是UDP不支持。

TCP将带外数据称为紧急数据（urgent
data）。TCP仅支持一个字节的紧急数据，但是允许紧急数据在普通数据传递机制数据流之外传输。

为了产生紧急数据，可以在3个send函数中的任何一个里指定MSG_OOB标志。如果带MSG_OOB标志发送的字节数超过一个时，最后一个字节将被视为紧急数据字节。

如果通过套接字安排了信号的产生，那么紧急数据被接收时，会发送SIGURG信号。

可以通过调用以下函数安排进程接收套接字的信号：

.. code:: c

   fcntl(sockfd, F_SETOWN, pid);

TCP支持紧急标记（urgent
mark）的概念，即在普通数据流中紧急数据所在的位置。如果采用套接字选项SO_OOBINLINE，那么可以在普通数据中接收紧急数据。

可以使用函数sockatmark来判断是否已经到达紧急标记：

.. code:: c

   #include <sys/socket.h>

   int sockatmark(int sockfd);
   // 返回值：若在标记处，返回1；若没在标记处，返回0；若出错，返回−1

当下一个要读取的字节在紧急标志处时，sockatmark返回1。

非阻塞和异步I/O
~~~~~~~~~~~~~~~

在基于套接字的异步I/O中，启动异步I/O有两个步骤：

1. 建立套接字所有权，这样信号可以被传递到合适的进程。
2. 通知套接字当I/O操作不会阻塞时发信号。

可以使用3种方式来完成第一个步骤。

1. 在fcntl中使用F_SETOWN命令。
2. 在ioctl中使用FIOSETOWN命令。
3. 在ioctl中使用SIOCSPGRP命令。

可以使用2种方式完成第二个步骤。

1. 在fcntl中使用F_SETFL命令并且启用文件标志O_ASYNC。
2. 在ioctl中使用FIOASYNC命令。
