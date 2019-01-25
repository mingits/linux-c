#  第 37 章 socket 编程

socket 这个词可以表示很多概念：

- 在TCP/IP协议中，「IP地址+TCP或UDP端口号」唯一标识网络通讯中的一个进程，「IP地址+端口号」就称为socket。
- 在TCP协议中，建立连接的两个进程各自有一个socket来标识，那么这两个socket组成的socket pair就唯一标识一个连接。socket本身有「插座」的意思，因此用来描述网络连接的一对一关系。
- TCP/IP协议最早在BSD UNIX上实现，为TCP/IP协议设计的应用层编程接口称为socket API。

本节的主要内容是socket API，主要介绍TCP协议的函数接口，最后简要介绍UDP协议和UNIX Domain Socket的函数接口。

## 1. 预备知识

### 1.1. 网络字节序

我们已经知道，内存中的多字节数据相对于内存地址有大端和小端之分，磁盘文件中的多字节数据相对于文件中的偏移地址也有大端小端之分。网络数据流同样有大端小端之分，那么如何定义网络数据流的地址呢？发送主机通常将发送缓冲区中的数据按内存地址从低到高的顺序发出，接收主机把从网络上接到的字节依次保存在接收缓冲区中，也是按内存地址从低到高的顺序保存，因此，网络数据流的地址应这样规定：先发出的数据是低地址，后发出的数据是高地址。

TCP/IP协议规定，网络数据流应采用大端字节序，即低地址高字节。例如上一节的UDP段格式，地址0-1是16位的源端口号，如果这个端口号是1000（0x3e8），则地址0是0x03，地址1是0xe8，也就是先发0x03，再发0xe8，这16位在发送主机的缓冲区中也应该是低地址存0x03，高地址存0xe8。但是，如果发送主机是小端字节序的，这16位被解释成0xe803，而不是1000。因此，发送主机把1000填到发送缓冲区之前需要做字节序的转换。同样地，接收主机如果是小端字节序的，接到16位的源端口号也要做字节序的转换。如果主机是大端字节序的，发送和接收都不需要做转换。同理，32位的IP地址也要考虑网络字节序和主机字节序的问题。

为使网络程序具有可移植性，使同样的C代码在大端和小端计算机上编译后都能正常运行，可以调用以下库函数做网络字节序和主机字节序的转换。

```
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```

这些函数名很好记，h表示host，n表示network，l表示32位长整数，s表示16位短整数。例如htonl表示将32位的长整数从主机字节序转换为网络字节序，例如将IP地址转换后准备发送。如果主机是小端字节序，这些函数将参数做相应的大小端转换然后返回，如果主机是大端字节序，这些函数不做转换，将参数原封不动地返回。

### 1.2. socket地址的数据类型及相关函数

socket API是一层抽象的网络编程接口，适用于各种底层网络协议，如IPv4、IPv6，以及后面要讲的UNIX Domain Socket。然而，各种网络协议的地址格式并不相同，如下图所示：

**图 37.1. sockaddr数据结构**

![sockaddr数据结构](../images/socket.sockaddr.png)

IPv4和IPv6的地址格式定义在`netinet/in.h`中，IPv4地址用sockaddr_in结构体表示，包括16位端口号和32位IP地址，IPv6地址用sockaddr_in6结构体表示，包括16位端口号、128位IP地址和一些控制字段。UNIX Domain Socket的地址格式定义在`sys/un.h`中，用sockaddr_un结构体表示。各种socket地址结构体的开头都是相同的，前16位表示整个结构体的长度（并不是所有UNIX的实现都有长度字段，如Linux就没有），后16位表示地址类型。IPv4、IPv6和UNIX  Domain  Socket的地址类型分别定义为常数AF_INET、AF_INET6、AF_UNIX。这样，只要取得某种sockaddr结构体的首地址，不需要知道具体是哪种类型的sockaddr结构体，就可以根据地址类型字段确定结构体中的内容。因此，socket  API可以接受各种类型的sockaddr结构体指针做参数，例如bind、accept、connect等函数，这些函数的参数应该设计成void  *类型以便接受各种类型的指针，但是sock API的实现早于ANSI C标准化，那时还没有void *类型，因此这些函数的参数都用struct  sockaddr *类型表示，在传递参数之前要强制类型转换一下，例如：

```
struct sockaddr_in servaddr;
/* initialize servaddr */
bind(listen_fd, (struct sockaddr *)&servaddr, sizeof(servaddr));
```

本节只介绍基于IPv4的socket网络编程，sockaddr_in中的成员struct in_addr sin_addr表示32位的IP地址。但是我们通常用点分十进制的字符串表示IP地址，以下函数可以在字符串表示和in_addr表示之间转换。

字符串转in_addr的函数：

```
#include <arpa/inet.h>

int inet_aton(const char *strptr, struct in_addr *addrptr);
in_addr_t inet_addr(const char *strptr);
int inet_pton(int family, const char *strptr, void *addrptr);
```

in_addr转字符串的函数：

```
char *inet_ntoa(struct in_addr inaddr);
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
```

其中inet_pton和inet_ntop不仅可以转换IPv4的in_addr，还可以转换IPv6的in6_addr，因此函数接口是void *addrptr。

## 2. 基于TCP协议的网络程序

下图是基于TCP协议的客户端/服务器程序的一般流程：

**图 37.2. TCP协议通讯流程**

![TCP协议通讯流程](../images/socket.tcpflowchart.png)

服务器调用socket()、bind()、listen()完成初始化后，调用accept()阻塞等待，处于监听端口的状态，客户端调用socket()初始化后，调用connect()发出SYN段并阻塞等待服务器应答，服务器应答一个SYN-ACK段，客户端收到后从connect()返回，同时应答一个ACK段，服务器收到后从accept()返回。

数据传输的过程：

建立连接后，TCP协议提供全双工的通信服务，但是一般的客户端/服务器程序的流程是由客户端主动发起请求，服务器被动处理请求，一问一答的方式。因此，服务器从accept()返回后立刻调用read()，读socket就像读管道一样，如果没有数据到达就阻塞等待，这时客户端调用write()发送请求给服务器，服务器收到后从read()返回，对客户端的请求进行处理，在此期间客户端调用read()阻塞等待服务器的应答，服务器调用write()将处理结果发回给客户端，再次调用read()阻塞等待下一条请求，客户端收到后从read()返回，发送下一条请求，如此循环下去。

如果客户端没有更多的请求了，就调用close()关闭连接，就像写端关闭的管道一样，服务器的read()返回0，这样服务器就知道客户端关闭了连接，也调用close()关闭连接。注意，任何一方调用close()后，连接的两个传输方向都关闭，不能再发送数据了。如果一方调用shutdown()则连接处于半关闭状态，仍可接收对方发来的数据。

在学习socket API时要注意应用程序和TCP协议层是如何交互的： *应用程序调用某个socket函数时TCP协议层完成什么动作，比如调用connect()会发出SYN段 *应用程序如何知道TCP协议层的状态变化，比如从某个阻塞的socket函数返回就表明TCP协议收到了某些段，再比如read()返回0就表明收到了FIN段

### 2.1. 最简单的TCP网络程序

下面通过最简单的客户端/服务器程序的实例来学习socket API。

server.c的作用是从客户端读字符，然后将每个字符转换为大写并回送给客户端。

```
/* server.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define MAXLINE 80
#define SERV_PORT 8000

int main(void)
{
	struct sockaddr_in servaddr, cliaddr;
	socklen_t cliaddr_len;
	int listenfd, connfd;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN];
	int i, n;

	listenfd = socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(SERV_PORT);
    
	bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	listen(listenfd, 20);

	printf("Accepting connections ...\n");
	while (1) {
		cliaddr_len = sizeof(cliaddr);
		connfd = accept(listenfd, 
				(struct sockaddr *)&cliaddr, &cliaddr_len);
	  
		n = read(connfd, buf, MAXLINE);
		printf("received from %s at PORT %d\n",
		       inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
		       ntohs(cliaddr.sin_port));
    
		for (i = 0; i < n; i++)
			buf[i] = toupper(buf[i]);
		write(connfd, buf, n);
		close(connfd);
	}
}
```

下面介绍程序中用到的socket API，这些函数都在`sys/socket.h`中。

```
int socket(int family, int type, int protocol);
```

socket()打开一个网络通讯端口，如果成功的话，就像open()一样返回一个文件描述符，应用程序可以像读写文件一样用read/write在网络上收发数据，如果socket()调用出错则返回-1。对于IPv4，family参数指定为AF_INET。对于TCP协议，type参数指定为SOCK_STREAM，表示面向流的传输协议。如果是UDP协议，则type参数指定为SOCK_DGRAM，表示面向数据报的传输协议。protocol参数的介绍从略，指定为0即可。

```
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

服务器程序所监听的网络地址和端口号通常是固定不变的，客户端程序得知服务器程序的地址和端口号后就可以向服务器发起连接，因此服务器需要调用bind绑定一个固定的网络地址和端口号。bind()成功返回0，失败返回-1。

bind()的作用是将参数sockfd和myaddr绑定在一起，使sockfd这个用于网络通讯的文件描述符监听myaddr所描述的地址和端口号。前面讲过，struct  sockaddr  *是一个通用指针类型，myaddr参数实际上可以接受多种协议的sockaddr结构体，而它们的长度各不相同，所以需要第三个参数addrlen指定结构体的长度。我们的程序中对myaddr参数是这样初始化的：

```
bzero(&servaddr, sizeof(servaddr));
servaddr.sin_family = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
servaddr.sin_port = htons(SERV_PORT);
```

首先将整个结构体清零，然后设置地址类型为AF_INET，网络地址为INADDR_ANY，这个宏表示本地的任意IP地址，因为服务器可能有多个网卡，每个网卡也可能绑定多个IP地址，这样设置可以在所有的IP地址上监听，直到与某个客户端建立了连接时才确定下来到底用哪个IP地址，端口号为SERV_PORT，我们定义为8000。

```
int listen(int sockfd, int backlog);
```

典型的服务器程序可以同时服务于多个客户端，当有客户端发起连接时，服务器调用的accept()返回并接受这个连接，如果有大量的客户端发起连接而服务器来不及处理，尚未accept的客户端就处于连接等待状态，listen()声明sockfd处于监听状态，并且最多允许有backlog个客户端处于连接待状态，如果接收到更多的连接请求就忽略。listen()成功返回0，失败返回-1。

```
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```

三方握手完成后，服务器调用accept()接受连接，如果服务器调用accept()时还没有客户端的连接请求，就阻塞等待直到有客户端连接上来。cliaddr是一个传出参数，accept()返回时传出客户端的地址和端口号。addrlen参数是一个传入传出参数（value-result   argument），传入的是调用者提供的缓冲区cliaddr的长度以避免缓冲区溢出问题，传出的是客户端地址结构体的实际长度（有可能没有占满调用者提供的缓冲区）。如果给cliaddr参数传NULL，表示不关心客户端的地址。

我们的服务器程序结构是这样的：

```
while (1) {
	cliaddr_len = sizeof(cliaddr);
	connfd = accept(listenfd, 
			(struct sockaddr *)&cliaddr, &cliaddr_len);
	n = read(connfd, buf, MAXLINE);
	...
	close(connfd);
}
```

整个是一个while死循环，每次循环处理一个客户端连接。由于cliaddr_len是传入传出参数，每次调用accept()之前应该重新赋初值。accept()的参数listenfd是先前的监听文件描述符，而accept()的返回值是另外一个文件描述符connfd，之后与客户端之间就通过这个connfd通讯，最后关闭connfd断开连接，而不关闭listenfd，再次回到循环开头listenfd仍然用作accept的参数。accept()成功返回一个文件描述符，出错返回-1。

client.c的作用是从命令行参数中获得一个字符串发给服务器，然后接收服务器返回的字符串并打印。

```
/* client.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define MAXLINE 80
#define SERV_PORT 8000

int main(int argc, char *argv[])
{
	struct sockaddr_in servaddr;
	char buf[MAXLINE];
	int sockfd, n;
	char *str;
    
	if (argc != 2) {
		fputs("usage: ./client message\n", stderr);
		exit(1);
	}
	str = argv[1];
    
	sockfd = socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);
	servaddr.sin_port = htons(SERV_PORT);
    
	connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	write(sockfd, str, strlen(str));

	n = read(sockfd, buf, MAXLINE);
	printf("Response from server:\n");
	write(STDOUT_FILENO, buf, n);

	close(sockfd);
	return 0;
}
```

由于客户端不需要固定的端口号，因此不必调用bind()，客户端的端口号由内核自动分配。注意，客户端不是不允许调用bind()，只是没有必要调用bind()固定一个端口号，服务器也不是必须调用bind()，但如果服务器不调用bind()，内核会自动给服务器分配监听端口，每次启动服务器时端口号都不一样，客户端要连接服务器就会遇到麻烦。

```
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```

客户端需要调用connect()连接服务器，connect和bind的参数形式一致，区别在于bind的参数是自己的地址，而connect的参数是对方的地址。connect()成功返回0，出错返回-1。

先编译运行服务器: 

```
$ ./server
 Accepting connections ...
```

然后在另一个终端里用netstat命令查看：

```
$ netstat -apn|grep 8000
 tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN     8148/server
```

可以看到server程序监听8000端口，IP地址还没确定下来。现在编译运行客户端：

```
$ ./client abcd
Response from server:
ABCD
```

回到server所在的终端，看看server的输出：

```
$ ./server
 Accepting connections ...
 received from 127.0.0.1 at PORT 59757
```

可见客户端的端口号是自动分配的。现在把客户端所连接的服务器IP改为其它主机的IP，试试两台主机的通讯。

再做一个小实验，在客户端的connect()代码之后插一个while(1);死循环，使客户端和服务器都处于连接中的状态，用netstat命令查看：

```
$ ./server &
[1] 8343
$ Accepting connections ...
./client abcd &
[2] 8344
$ netstat -apn|grep 8000
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN     8343/server         
tcp        0      0 127.0.0.1:44406         127.0.0.1:8000          ESTABLISHED8344/client         
tcp        0      0 127.0.0.1:8000          127.0.0.1:44406         ESTABLISHED8343/server
```

应用程序中的一个socket文件描述符对应一个socket pair，也就是源地址:源端口号和目的地址:目的端口号，也对应一个TCP连接。

**表 37.1. client和server的socket状态**

| socket文件描述符     | 源地址:源端口号 | 目的地址:目的端口号 | 状态        |
| -------------------- | --------------- | ------------------- | ----------- |
| server.c中的listenfd | 0.0.0.0:8000    | 0.0.0.0:*           | LISTEN      |
| server.c中的connfd   | 127.0.0.1:8000  | 127.0.0.1:44406     | ESTABLISHED |
| client.c中的sockfd   | 127.0.0.1:44406 | 127.0.0.1:8000      | ESTABLISHED |

### 2.2. 错误处理与读写控制

上面的例子不仅功能简单，而且简单到几乎没有什么错误处理，我们知道，系统调用不能保证每次都成功，必须进行出错处理，这样一方面可以保证程序逻辑正常，另一方面可以迅速得到故障信息。

为使错误处理的代码不影响主程序的可读性，我们把与socket相关的一些系统函数加上错误处理代码包装成新的函数，做成一个模块wrap.c：

```
#include <stdlib.h>
#include <errno.h>
#include <sys/socket.h>

void perr_exit(const char *s)
{
	perror(s);
	exit(1);
}

int Accept(int fd, struct sockaddr *sa, socklen_t *salenptr)
{
	int n;

again:
	if ( (n = accept(fd, sa, salenptr)) < 0) {
		if ((errno == ECONNABORTED) || (errno == EINTR))
			goto again;
		else
			perr_exit("accept error");
	}
	return n;
}

void Bind(int fd, const struct sockaddr *sa, socklen_t salen)
{
	if (bind(fd, sa, salen) < 0)
		perr_exit("bind error");
}

void Connect(int fd, const struct sockaddr *sa, socklen_t salen)
{
	if (connect(fd, sa, salen) < 0)
		perr_exit("connect error");
}

void Listen(int fd, int backlog)
{
	if (listen(fd, backlog) < 0)
		perr_exit("listen error");
}

int Socket(int family, int type, int protocol)
{
	int n;

	if ( (n = socket(family, type, protocol)) < 0)
		perr_exit("socket error");
	return n;
}

ssize_t Read(int fd, void *ptr, size_t nbytes)
{
	ssize_t n;

again:
	if ( (n = read(fd, ptr, nbytes)) == -1) {
		if (errno == EINTR)
			goto again;
		else
			return -1;
	}
	return n;
}

ssize_t Write(int fd, const void *ptr, size_t nbytes)
{
	ssize_t n;

again:
	if ( (n = write(fd, ptr, nbytes)) == -1) {
		if (errno == EINTR)
			goto again;
		else
			return -1;
	}
	return n;
}

void Close(int fd)
{
	if (close(fd) == -1)
		perr_exit("close error");
}
```

慢系统调用accept、read和write被信号中断时应该重试。connect虽然也会阻塞，但是被信号中断时不能立刻重试。对于accept，如果errno是ECONNABORTED，也应该重试。详细解释见参考资料。

TCP协议是面向流的，read和write调用的返回值往往小于参数指定的字节数。对于read调用，如果接收缓冲区中有20字节，请求读100个字节，就会返回20。对于write调用，如果请求写100个字节，而发送缓冲区中只有20个字节的空闲位置，那么write会阻塞，直到把100个字节全部交给发送缓冲区才返回，但如果socket文件描述符有O_NONBLOCK标志，则write不阻塞，直接返回20。为避免这些情况干扰主程序的逻辑，确保读写我们所请求的字节数，我们实现了两个包装函数readn和writen，也放在wrap.c中：

```
ssize_t Readn(int fd, void *vptr, size_t n)
{
	size_t  nleft;
	ssize_t nread;
	char   *ptr;

	ptr = vptr;
	nleft = n;
	while (nleft > 0) {
		if ( (nread = read(fd, ptr, nleft)) < 0) {
			if (errno == EINTR)
				nread = 0;
			else
				return -1;
		} else if (nread == 0)
			break;

		nleft -= nread;
		ptr += nread;
	}
	return n - nleft;
}

ssize_t Writen(int fd, const void *vptr, size_t n)
{
	size_t nleft;
	ssize_t nwritten;
	const char *ptr;

	ptr = vptr;
	nleft = n;
	while (nleft > 0) {
		if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
			if (nwritten < 0 && errno == EINTR)
				nwritten = 0;
			else
				return -1;
		}

		nleft -= nwritten;
		ptr += nwritten;
	}
	return n;
}
```

如果应用层协议的各字段长度固定，用readn来读是非常方便的。例如设计一种客户端上传文件的协议，规定前12字节表示文件名，超过12字节的文件名截断，不足12字节的文件名用'\0'补齐，从第13字节开始是文件内容，上传完所有文件内容后关闭连接，服务器可以先调用readn读12个字节，根据文件名创建文件，然后在一个循环中调用read读文件内容并存盘，循环结束的条件是read返回0。

字段长度固定的协议往往不够灵活，难以适应新的变化。比如，以前DOS的文件名是8字节主文件名加「.」加3字节扩展名，不超过12字节，但是现代操作系统的文件名可以长得多，12字节就不够用了。那么制定一个新版本的协议规定文件名字段为256字节怎么样？这样又造成很大的浪费，因为大多数文件名都很短，需要用大量的'\0'补齐256字节，而且新版本的协议和老版本的程序无法兼容，如果已经有很多人在用老版本的程序了，会造成遵循新协议的程序与老版本程序的互操作性（Interoperability）问题。如果新版本的协议要添加新的字段，比如规定前12字节是文件名，从13到16字节是文件类型说明，从第17字节开始才是文件内容，同样会造成和老版本的程序无法兼容的问题。

现在重新看看上一节的TFTP协议是如何避免上述问题的：TFTP协议的各字段是可变长的，以'\0'为分隔符，文件名可以任意长，再看blksize等几个选项字段，TFTP协议并没有规定从第m字节到第n字节是blksize的值，而是把选项的描述信息「blksize」与它的值「512」一起做成一个可变长的字段，这样，以后添加新的选项仍然可以和老版本的程序兼容（老版本的程序只要忽略不认识的选项就行了）。

因此，常见的应用层协议都是带有可变长字段的，字段之间的分隔符用换行的比用'\0'的更常见，例如本节后面要介绍的HTTP协议。可变长字段的协议用readn来读就很不方便了，为此我们实现一个类似于fgets的readline函数，也放在wrap.c中：

```
static ssize_t my_read(int fd, char *ptr)
{
	static int read_cnt;
	static char *read_ptr;
	static char read_buf[100];

	if (read_cnt <= 0) {
	again:
		if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
			if (errno == EINTR)
				goto again;
			return -1;
		} else if (read_cnt == 0)
			return 0;
		read_ptr = read_buf;
	}
	read_cnt--;
	*ptr = *read_ptr++;
	return 1;
}

ssize_t Readline(int fd, void *vptr, size_t maxlen)
{
	ssize_t n, rc;
	char    c, *ptr;

	ptr = vptr;
	for (n = 1; n < maxlen; n++) {
		if ( (rc = my_read(fd, &c)) == 1) {
			*ptr++ = c;
			if (c  == '\n')
				break;
		} else if (rc == 0) {
			*ptr = 0;
			return n - 1;
		} else
			return -1;
	}
	*ptr  = 0;
	return n;
}
```

#### 习题

1、请读者自己写出wrap.c的头文件wrap.h，后面的网络程序代码都要用到这个头文件。

2、修改server.c和client.c，添加错误处理。

### 2.3. 把client改为交互式输入

目前实现的client每次运行只能从命令行读取一个字符串发给服务器，再从服务器收回来，现在我们把它改成交互式的，不断从终端接受用户输入并和server交互。 

```
/* client.c */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include "wrap.h"

#define MAXLINE 80
#define SERV_PORT 8000

int main(int argc, char *argv[])
{
	struct sockaddr_in servaddr;
	char buf[MAXLINE];
	int sockfd, n;
    
	sockfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);
	servaddr.sin_port = htons(SERV_PORT);
    
	Connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	while (fgets(buf, MAXLINE, stdin) != NULL) {
		Write(sockfd, buf, strlen(buf));
		n = Read(sockfd, buf, MAXLINE);
		if (n == 0)
			printf("the other side has been closed.\n");
		else
			Write(STDOUT_FILENO, buf, n);
	}

	Close(sockfd);
	return 0;
}
```

编译并运行server和client，看看是否达到了你预想的结果。 

```
$ ./client
haha1
HAHA1 
haha2
the other side has been closed.
haha3
$
```

这时server仍在运行，但是client的运行结果并不正确。原因是什么呢？仔细查看server.c可以发现，server对每个请求只处理一次，应答后就关闭连接，client不能继续使用这个连接发送数据。但是client下次循环时又调用write发数据给server，write调用只负责把数据交给TCP发送缓冲区就可以成功返回了，所以不会出错，而server收到数据后应答一个RST段，client收到RST段后无法立刻通知应用层，只把这个状态保存在TCP协议层。client下次循环又调用write发数据给server，由于TCP协议层已经处于RST状态了，因此不会将数据发出，而是发一个SIGPIPE信号给应用层，SIGPIPE信号的缺省处理动作是终止程序，所以看到上面的现象。

为了避免client异常退出，上面的代码应该在判断对方关闭了连接后break出循环，而不是继续write。另外，有时候代码中需要连续多次调用write，可能还来不及调用read得知对方已关闭了连接就被SIGPIPE信号终止掉了，这就需要在初始化时调用sigaction处理SIGPIPE信号，如果SIGPIPE信号没有导致进程异常退出，write返回-1并且errno为EPIPE。

另外，我们需要修改server，使它可以多次处理同一客户端的请求。

```
/* server.c */
#include <stdio.h>
#include <string.h>
#include <netinet/in.h>
#include "wrap.h"

#define MAXLINE 80
#define SERV_PORT 8000

int main(void)
{
	struct sockaddr_in servaddr, cliaddr;
	socklen_t cliaddr_len;
	int listenfd, connfd;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN];
	int i, n;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(SERV_PORT);
    
	Bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	Listen(listenfd, 20);

	printf("Accepting connections ...\n");
	while (1) {
		cliaddr_len = sizeof(cliaddr);
		connfd = Accept(listenfd, 
				(struct sockaddr *)&cliaddr, &cliaddr_len);
		while (1) {
			n = Read(connfd, buf, MAXLINE);
			if (n == 0) {
				printf("the other side has been closed.\n");
				break;
			}
			printf("received from %s at PORT %d\n",
			       inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
			       ntohs(cliaddr.sin_port));
    
			for (i = 0; i < n; i++)
				buf[i] = toupper(buf[i]);
			Write(connfd, buf, n);
		}
		Close(connfd);
	}
}
```

经过上面的修改后，客户端和服务器可以进行多次交互了。我们知道，服务器通常是要同时服务多个客户端的，运行上面的server和client之后，再开一个终端运行client试试，新的client能得到服务吗？想想为什么。

### 2.4. 使用fork并发处理多个client的请求

怎么解决这个问题？网络服务器通常用fork来同时服务多个客户端，父进程专门负责监听端口，每次accept一个新的客户端连接就fork出一个子进程专门服务这个客户端。但是子进程退出时会产生僵尸进程，父进程要注意处理SIGCHLD信号和调用wait清理僵尸进程。

以下给出代码框架，完整的代码请读者自己完成。

```
listenfd = socket(...);
bind(listenfd, ...);
listen(listenfd, ...); 
while (1) {
	connfd = accept(listenfd, ...);
	n = fork();
	if (n == -1) {
		perror("call to fork");
		exit(1);
	} else if (n == 0) {
		close(listenfd);
		while (1) {
			read(connfd, ...);
			...
			write(connfd, ...);
		}
		close(connfd);
		exit(0);
	} else
		close(connfd);
}
```

### 2.5. setsockopt

现在做一个测试，首先启动server，然后启动client，然后用Ctrl-C使server终止，这时马上再运行server，结果是：

```
$ ./server
 bind error: Address already in use
```

这是因为，虽然server的应用程序终止了，但TCP协议层的连接并没有完全断开，因此不能再次监听同样的server端口。我们用netstat命令查看一下：

```
$ netstat -apn |grep 8000
 tcp        1      0 127.0.0.1:33498         127.0.0.1:8000          CLOSE_WAIT 10830/client        
 tcp        0      0 127.0.0.1:8000          127.0.0.1:33498         FIN_WAIT2  -
```

server终止时，socket描述符会自动关闭并发FIN段给client，client收到FIN后处于CLOSE_WAIT状态，但是client并没有终止，也没有关闭socket描述符，因此不会发FIN给server，因此server的TCP连接处于FIN_WAIT2状态。

现在用Ctrl-C把client也终止掉，再观察现象：

```
$ netstat -apn |grep 8000
 tcp        0      0 127.0.0.1:8000          127.0.0.1:44685         TIME_WAIT  -
 $ ./server
 bind error: Address already in use
```

client终止时自动关闭socket描述符，server的TCP连接收到client发的FIN段后处于TIME_WAIT状态。TCP协议规定，主动关闭连接的一方要处于TIME_WAIT状态，等待两个MSL（maximum  segment  lifetime）的时间后才能回到CLOSED状态，因为我们先Ctrl-C终止了server，所以server是主动关闭连接的一方，在TIME_WAIT期间仍然不能再次监听同样的server端口。MSL在RFC1122中规定为两分钟，但是各操作系统的实现不同，在Linux上一般经过半分钟后就可以再次启动server了。至于为什么要规定TIME_WAIT的时间请读者参考UNP  2.7节。

在server的TCP连接没有完全断开之前不允许重新监听是不合理的，因为，TCP连接没有完全断开指的是connfd（127.0.0.1:8000）没有完全断开，而我们重新监听的是listenfd（0.0.0.0:8000），虽然是占用同一个端口，但IP地址不同，connfd对应的是与某个客户端通讯的一个具体的IP地址，而listenfd对应的是wildcard   address。解决这个问题的方法是使用setsockopt()设置socket描述符的选项SO_REUSEADDR为1，表示允许创建端口号相同但IP地址不同的多个socket描述符。在server代码的socket()和bind()调用之间插入如下代码：

```
int opt = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

有关setsockopt可以设置的其它选项请参考UNP第7章。

### 2.6. 使用select

select是网络程序中很常用的一个系统调用，它可以同时监听多个阻塞的文件描述符（例如多个网络连接），哪个有数据到达就处理哪个，这样，不需要fork和多进程就可以实现并发服务的server。

```
/* server.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netinet/in.h>
#include "wrap.h"

#define MAXLINE 80
#define SERV_PORT 8000

int main(int argc, char **argv)
{
	int i, maxi, maxfd, listenfd, connfd, sockfd;
	int nready, client[FD_SETSIZE];
	ssize_t n;
	fd_set rset, allset;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN];
	socklen_t cliaddr_len;
	struct sockaddr_in	cliaddr, servaddr;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(SERV_PORT);

	Bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	Listen(listenfd, 20);

	maxfd = listenfd;		/* initialize */
	maxi = -1;			/* index into client[] array */
	for (i = 0; i < FD_SETSIZE; i++)
		client[i] = -1;	/* -1 indicates available entry */
	FD_ZERO(&allset);
	FD_SET(listenfd, &allset);

	for ( ; ; ) {
		rset = allset;	/* structure assignment */
		nready = select(maxfd+1, &rset, NULL, NULL, NULL);
		if (nready < 0)
			perr_exit("select error");

		if (FD_ISSET(listenfd, &rset)) { /* new client connection */
			cliaddr_len = sizeof(cliaddr);
			connfd = Accept(listenfd, (struct sockaddr *)&cliaddr, &cliaddr_len);

			printf("received from %s at PORT %d\n",
			       inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
			       ntohs(cliaddr.sin_port));

			for (i = 0; i < FD_SETSIZE; i++)
				if (client[i] < 0) {
					client[i] = connfd; /* save descriptor */
					break;
				}
			if (i == FD_SETSIZE) {
				fputs("too many clients\n", stderr);
				exit(1);
			}

			FD_SET(connfd, &allset);	/* add new descriptor to set */
			if (connfd > maxfd)
				maxfd = connfd; /* for select */
			if (i > maxi)
				maxi = i;	/* max index in client[] array */

			if (--nready == 0)
				continue;	/* no more readable descriptors */
		}

		for (i = 0; i <= maxi; i++) {	/* check all clients for data */
			if ( (sockfd = client[i]) < 0)
				continue;
			if (FD_ISSET(sockfd, &rset)) {
				if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
					/* connection closed by client */
					Close(sockfd);
					FD_CLR(sockfd, &allset);
					client[i] = -1;
				} else {
					int j;
					for (j = 0; j < n; j++)
						buf[j] = toupper(buf[j]);
					Write(sockfd, buf, n);
				}

				if (--nready == 0)
					break;	/* no more readable descriptors */
			}
		}
	}
}
```

## 3. 基于UDP协议的网络程序

下图是典型的UDP客户端/服务器通讯过程（该图出自[[UNPv13e\]](bi01.html#bibli.unp)）。

**图 37.3. UDP通讯流程**

![UDP通讯流程](../images/socket.udpflowchart.png)

以下是简单的UDP服务器和客户端程序。

```
/* server.c */
#include <stdio.h>
#include <string.h>
#include <netinet/in.h>
#include "wrap.h"

#define MAXLINE 80
#define SERV_PORT 8000

int main(void)
{
	struct sockaddr_in servaddr, cliaddr;
	socklen_t cliaddr_len;
	int sockfd;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN];
	int i, n;

	sockfd = Socket(AF_INET, SOCK_DGRAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(SERV_PORT);
    
	Bind(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	printf("Accepting connections ...\n");
	while (1) {
		cliaddr_len = sizeof(cliaddr);
		n = recvfrom(sockfd, buf, MAXLINE, 0, (struct sockaddr *)&cliaddr, &cliaddr_len);
		if (n == -1)
			perr_exit("recvfrom error");
		printf("received from %s at PORT %d\n",
		       inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
		       ntohs(cliaddr.sin_port));
    
		for (i = 0; i < n; i++)
			buf[i] = toupper(buf[i]);
		n = sendto(sockfd, buf, n, 0, (struct sockaddr *)&cliaddr, sizeof(cliaddr));
		if (n == -1)
			perr_exit("sendto error");
	}
}
/* client.c */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include "wrap.h"

#define MAXLINE 80
#define SERV_PORT 8000

int main(int argc, char *argv[])
{
	struct sockaddr_in servaddr;
	int sockfd, n;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN];
	socklen_t servaddr_len;
    
	sockfd = Socket(AF_INET, SOCK_DGRAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);
	servaddr.sin_port = htons(SERV_PORT);
    
	while (fgets(buf, MAXLINE, stdin) != NULL) {
		n = sendto(sockfd, buf, strlen(buf), 0, (struct sockaddr *)&servaddr, sizeof(servaddr));
		if (n == -1)
			perr_exit("sendto error");

		n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, 0);
		if (n == -1)
			perr_exit("recvfrom error");
	  
		Write(STDOUT_FILENO, buf, n);
	}

	Close(sockfd);
	return 0;
}
```

由于UDP不需要维护连接，程序逻辑简单了很多，但是UDP协议是不可靠的，实际上有很多保证通讯可靠性的机制需要在应用层实现。

编译运行server，在两个终端里各开一个client与server交互，看看server是否具有并发服务的能力。用Ctrl+C关闭server，然后再运行server，看此时client还能否和server联系上。和前面TCP程序的运行结果相比较，体会无连接的含义。

## 4. UNIX Domain Socket IPC

socket  API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain  Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain  Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX  Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain  Socket也是可靠的，消息既不会丢失也不会顺序错乱。

UNIX Domain Socket是全双工的，API接口语义丰富，相比其它IPC机制有明显的优越性，目前已成为使用最广泛的IPC机制，比如X Window服务器和GUI程序之间就是通过UNIX Domain Socket通讯的。

使用UNIX  Domain Socket的过程和网络socket十分相似，也要先调用socket()创建一个socket文件描述符，address  family指定为AF_UNIX，type可以选择SOCK_DGRAM或SOCK_STREAM，protocol参数仍然指定为0即可。

UNIX  Domain  Socket与网络socket编程最明显的不同在于地址格式不同，用结构体sockaddr_un表示，网络编程的socket地址是IP地址加端口号，而UNIX  Domain  Socket的地址是一个socket类型的文件在文件系统中的路径，这个socket文件由bind()调用创建，如果调用bind()时该文件已存在，则bind()错误返回。

以下程序将UNIX Domain socket绑定到一个地址。

```
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <sys/socket.h>
#include <sys/un.h>

int main(void)
{
	int fd, size;
	struct sockaddr_un un;

	memset(&un, 0, sizeof(un));
	un.sun_family = AF_UNIX;
	strcpy(un.sun_path, "foo.socket");
	if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0) {
		perror("socket error");
		exit(1);
	}
	size = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);
	if (bind(fd, (struct sockaddr *)&un, size) < 0) {
		perror("bind error");
		exit(1);
	}
	printf("UNIX domain socket bound\n");
	exit(0);
}
```

注意程序中的offsetof宏，它在stddef.h头文件中定义：

```
#define offsetof(TYPE, MEMBER) ((int)&((TYPE *)0)->MEMBER)
```

offsetof(struct sockaddr_un, sun_path)就是取sockaddr_un结构体的sun_path成员在结构体中的偏移，也就是从结构体的第几个字节开始是sun_path成员。想一想，这个宏是如何实现这一功能的？

该程序的运行结果如下。

```
$ ./a.out
UNIX domain socket bound
$ ls -l foo.socket
srwxrwxr-x 1 user        0 Aug 22 12:43 foo.socket
$ ./a.out
bind error: Address already in use
$ rm foo.socket
$ ./a.out
UNIX domain socket bound
```

以下是服务器的listen模块，与网络socket编程类似，在bind之后要listen，表示通过bind的地址（也就是socket文件）提供服务。

```
#include <stddef.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <errno.h>

#define QLEN 10

/*
 * Create a server endpoint of a connection.
 * Returns fd if all OK, <0 on error.
 */
int serv_listen(const char *name)
{
	int                 fd, len, err, rval;
	struct sockaddr_un  un;

	/* create a UNIX domain stream socket */
	if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0)
		return(-1);
	unlink(name);   /* in case it already exists */

	/* fill in socket address structure */
	memset(&un, 0, sizeof(un));
	un.sun_family = AF_UNIX;
	strcpy(un.sun_path, name);
	len = offsetof(struct sockaddr_un, sun_path) + strlen(name);

	/* bind the name to the descriptor */
	if (bind(fd, (struct sockaddr *)&un, len) < 0) {
		rval = -2;
		goto errout;
	}
	if (listen(fd, QLEN) < 0) { /* tell kernel we're a server */
		rval = -3;
		goto errout;
	}
	return(fd);

errout:
	err = errno;
	close(fd);
	errno = err;
	return(rval);
}
```

以下是服务器的accept模块，通过accept得到客户端地址也应该是一个socket文件，如果不是socket文件就返回错误码，如果是socket文件，在建立连接后这个文件就没有用了，调用unlink把它删掉，通过传出参数uidptr返回客户端程序的user  id。

```
#include <stddef.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <errno.h>

int serv_accept(int listenfd, uid_t *uidptr)
{
	int                 clifd, len, err, rval;
	time_t              staletime;
	struct sockaddr_un  un;
	struct stat         statbuf;

	len = sizeof(un);
	if ((clifd = accept(listenfd, (struct sockaddr *)&un, &len)) < 0)
		return(-1);     /* often errno=EINTR, if signal caught */

	/* obtain the client's uid from its calling address */
	len -= offsetof(struct sockaddr_un, sun_path); /* len of pathname */
	un.sun_path[len] = 0;           /* null terminate */

	if (stat(un.sun_path, &statbuf) < 0) {
		rval = -2;
		goto errout;
	}

	if (S_ISSOCK(statbuf.st_mode) == 0) {
		rval = -3;      /* not a socket */
		goto errout;
	}

	if (uidptr != NULL)
		*uidptr = statbuf.st_uid;   /* return uid of caller */
	unlink(un.sun_path);        /* we're done with pathname now */
	return(clifd);

errout:
	err = errno;
	close(clifd);
	errno = err;
	return(rval);
}
```

以下是客户端的connect模块，与网络socket编程不同的是，UNIX Domain  Socket客户端一般要显式调用bind函数，而不依赖系统自动分配的地址。客户端bind一个自己指定的socket文件名的好处是，该文件名可以包含客户端的pid以便服务器区分不同的客户端。

```
#include <stdio.h>
#include <stddef.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <errno.h>

#define CLI_PATH    "/var/tmp/"      /* +5 for pid = 14 chars */

/*
 * Create a client endpoint and connect to a server.
 * Returns fd if all OK, <0 on error.
 */
int cli_conn(const char *name)
{
	int                fd, len, err, rval;
	struct sockaddr_un un;

	/* create a UNIX domain stream socket */
	if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0)
		return(-1);

	/* fill socket address structure with our address */
	memset(&un, 0, sizeof(un));
	un.sun_family = AF_UNIX;
	sprintf(un.sun_path, "%s%05d", CLI_PATH, getpid());
	len = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);

	unlink(un.sun_path);        /* in case it already exists */
	if (bind(fd, (struct sockaddr *)&un, len) < 0) {
		rval = -2;
		goto errout;
	}

	/* fill socket address structure with server's address */
	memset(&un, 0, sizeof(un));
	un.sun_family = AF_UNIX;
	strcpy(un.sun_path, name);
	len = offsetof(struct sockaddr_un, sun_path) + strlen(name);
	if (connect(fd, (struct sockaddr *)&un, len) < 0) {
		rval = -4;
		goto errout;
	}
	return(fd);

errout:
	err = errno;
	close(fd);
	errno = err;
	return(rval);
}
```

下面是自己动手时间，请利用以上模块编写完整的客户端/服务器通讯的程序。

## 5. 练习：实现简单的Web服务器

实现一个简单的Web服务器myhttpd。服务器程序启动时要读取配置文件/etc/myhttpd.conf，其中需要指定服务器监听的端口号和服务目录，例如：

```
Port=80
Directory=/var/www
```

注意，1024以下的端口号需要超级用户才能开启服务。如果你的系统中已经安装了某种Web服务器（例如Apache），应该为myhttpd选择一个不同的端口号。当浏览器向服务器请求文件时，服务器就从服务目录（例如/var/www）中找出这个文件，加上HTTP协议头一起发给浏览器。但是，如果浏览器请求的文件是可执行的则称为CGI程序，服务器并不是将这个文件发给浏览器，而是在服务器端执行这个程序，将它的标准输出发给浏览器，服务器不发送完整的HTTP协议头，CGI程序自己负责输出一部分HTTP协议头。

### 5.1. 基本HTTP协议

打开浏览器，输入服务器IP，例如 http://192.168.0.3 ，如果端口号不是80，例如是8000，则输入 http://192.168.0.3:8000 。这时浏览器向服务器发送的HTTP协议头如下：

```
GET / HTTP/1.1
Host: 192.168.0.3:8000
User-Agent: Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.8.1.6) Gecko/20061201 Firefox/2.0.0.6 (Ubuntu-feisty)
Accept: text/xml,application/xml,application/xhtml+xml,text/html;q=0.9,text/plain;q=0.8,image/png,*/*;q=0.5
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive: 300
Connection: keep-alive
```

注意，其中每一行的末尾都是回车加换行（C语言的"\r\n"），第一行是GET请求和协议版本，其余几行选项字段我们不讨论，*HTTP协议头的最后有一个空行，也是回车加换行*。

我们实现的Web服务器只要能正确解析第一行就行了，这是一个GET请求，请求的是服务目录的根目录/（在本例中实际上是/var/www），Web服务器应该把该目录下的索引页（默认是index.html）发给浏览器，也就是把/var/www/index.html发给浏览器。假如该文件的内容如下（HTML文件没必要以"\r\n"换行，以"\n"换行就可以了）：

```
<html>
<head><title>Test Page</title></head>
<body>
	<p>Test OK</p>
	<img src='mypic.jpg'>
</body>
</html>
```

显示一行字和一幅图片，图片的相对路径（相对当前的index.html文件的路径）是mypic.jpg，也就是/var/www/mypic.jpg，如果用绝对路径表示应该是：

```
<img src='/mypic.jpg'>
```

服务器应按如下格式应答浏览器：

```
HTTP/1.1 200 OK
Content-Type: text/html

<html>
<head><title>Test Page</title></head>
<body>
	<p>Test OK</p>
	<img src='mypic.jpg'>
</body>
</html>
```

服务器应答的HTTP头也是每行末尾以回车加换行结束，最后跟一个空行的回车加换行。

HTTP头的第一行是协议版本和应答码，200表示成功，后面的消息OK其实可以随意写，浏览器是不关心的，主要是为了调试时给开发人员看的。虽然网络协议最终是程序与程序之间的对话，但是在开发过程中却是人与程序之间的对话，一个设计透明的网络协议可以提供很多直观的信息给开发人员，因此，很多应用层网络协议，如HTTP、FTP、SMTP、POP3等都是基于文本的协议，为的是透明性（transparency）。

HTTP头的第二行表示即将发送的文件的类型（称为MIME类型），这里是text/html，纯文本文件是text/plain，图片则是image/jpg、image/png等。

然后就发送文件的内容，发送完毕之后主动关闭连接，这样浏览器就知道文件发送完了。这一点比较特殊：通常网络通信都是客户端主动发起连接，主动发起请求，主动关闭连接，服务器只是被动地处理各种情况，而HTTP协议规定服务器主动关闭连接（有些Web服务器可以配置成Keep-Alive的，我们不讨论这种情况）。

浏览器收到index.html之后，发现其中有一个图片文件，就会再发一个GET请求（HTTP协议头其余部分略）：

```
GET /mypic.jpg HTTP/1.1
```

一个较大的网页中可能有很多图片，浏览器可能在下载网页的同时就开很多线程下载图片，因此，'''服务器即使对同一个客户端也需要提供并行服务的能力'''。服务器收到这个请求应该把图片发过去然后关闭连接：

```
HTTP/1.1 200 OK
Content-Type: image/jpg

（这里是mypic.jpg的二进制数据）
```

这时浏览器就应该显示出完整的网页了。

如果浏览器请求的文件在服务器上找不到，要应答一个404错误页面，例如：

```
HTTP/1.1 404 Not Found
Content-Type: text/html

<html><body>request file not found</body></html>
```

### 5.2. 执行CGI程序

如果浏览器请求的是一个可执行文件（不管是什么样的可执行文件，即使是shell脚本也一样），那么服务器并不把这个文件本身发给浏览器，而是把它的执行结果标准输出发给浏览器。例如一个shell脚本/var/www/myscript.sh（注意一定要加可执行权限）：

```
#!/bin/sh
echo "Content-Type: text/html"
echo
echo "<html><body>Hello world!</body></html>"
```

这样浏览器收到的是：

```
HTTP/1.1 200 OK
Content-Type: text/html

<html><body>Hello world!</body></html>
```

总结一下服务器的处理步骤：

1. 解析浏览器的请求，在服务目录中查找相应的文件，如果找不到该文件就返回404错误页面
2. 如果找到了浏览器请求的文件，用stat(2)检查它是否可执行
3. 如果该文件可执行：
   1. 发送HTTP/1.1 200 OK给客户端
   2. fork(2)，然后用dup2(2)重定向子进程的标准输出到客户端socket
   3. 在子进程中exec(3)该CGI程序
   4. 关闭连接
4. 如果该文件不可执行：
   1. 发送HTTP/1.1 200 OK给客户端
   2. 如果是一个图片文件，根据图片的扩展名发送相应的Content-Type给客户端
   3. 如果不是图片文件，这里我们简化处理，都当作Content-Type: text/html
   4. 简单的HTTP协议头有这两行就足够了，再发一个空行表示结束
   5. 读取文件的内容发送到客户端
   6. 关闭连接