---
title: 『Linux高性能服务器编程』读书笔记
key: Linux高性能服务器编程读书笔记
---

<!--more-->

## 0x05 基础API

```c++
//1. 主机序和网络字节序转换
#include <netinet/in.h>
unsigned long int htonl (unsigned long int hostlong); // host to network long
unsigned short int htons (unsigned short int hostlong); // host to network short

unsigned long int ntonhl (unsigned long int netlong);//network to host long 
unsigned short int ntohs (unsigned short int netlong);//network to host short 
  
//2. socket地址
//2.1 通用socket结构 sockaddr
struct sockaddr
  {
    sa_family_t sa_family;	/* 地址族，AF_XXX  */
    char sa_data[14];		/* Address data.包含目的地址和端口信息  */
  };
/*
  sa_family: 2字节地址族：
    AF_UNIX :UNIX本地地址族
    AF_INET :TCP/IPv4地址族
    AF_INET6:TCP/IPv6地址族
*/

//2.2 专用socket结构 sockaddr_in，与前者相比把port和addr分开存储到了两个变量中
struct sockaddr_in
  {
    sa_family_t sin_family;     /* 地址族，AF_XXX  */
    in_port_t sin_port;			/* 端口号，要用网络字节序*/
    struct in_addr sin_addr;	/* IPv4地址结构体，见↓ */
  };
//存储IPv4的数据结构
struct in_addr
  {
    uint32_t s_addr; /* IPv4地址，要用网络字节序表示*/
  };

//3. IP地址转换函数
#include <arpa/inet.h>
// 将点分十进制字符串的IPv4地址, 转换为网络字节序整数表示的IPv4地址. 失败返回INADDR_NONE
in_addr_t inet_addr( const char* strptr);
/*例如：*/ sockaddr_in.sin_addr.s_addr=inet_addr("127.0.0.1");

// 功能相同不过转换结果存在 inp指向的结构体中. 成功返回1 反之返回0
int inet_aton( const char* cp, struct in_addr* inp);

//将IPv4地址转化为点分十进制IP地址字符串，和上述两个函数功能相反
//该函数使用一个内部静态存储转化结果，函数返回值指向该静态内存，所以此函数是不可重入的！
char* inet_ntoa(struct in_addr in); 


//4. 创建socket
int socket(int domain, int type, int protocol);
/*
  domain:底层协议族
    PF_UNIX :UNIX本地协议族
    PF_INET :TCP/IPv4协议族
    PF_INET6:TCP/IPv6协议族
    (地址族和协议族事实上是一一对应的，在前UNIX时代有所区分，在Linux上已完全兼容，不过为了规范区分一下也行)
  type:通讯类型(服务类型)
    SOCK_STREAM(流服务，TCP协议)
    SOCK_DGRAM(数据报服务，UDP类型)
  protocol:通常为0
  socket调用成功时返回一个socket文件描述符，失败返回-1并设置errno
*/

//5. 命名socket
```



## 0x06 高级I/O函数

## 0X09 I/O复用