<!-- TOC -->

- [1. 网络相关](#1-网络相关)
    - [1.1. 地址相关](#11-地址相关)
        - [1.1.1. sockaddr](#111-sockaddr)
        - [1.1.2. sockaddr_in](#112-sockaddr_in)
        - [1.1.3. in_addr](#113-in_addr)
    - [1.2. 字符串地址与in_addr地址的相互转换](#12-字符串地址与in_addr地址的相互转换)

<!-- /TOC -->
# 1. 网络相关
## 1.1. 地址相关
### 1.1.1. sockaddr
```c
struct sockaddr {
    unsigned short sa_family;  /*2字节，地址家族*/ 
    char sa_data[14];  /*14字节，协议地址*/
};
```
### 1.1.2. sockaddr_in
```c
struct sockaddr_in {
    short int sin_family;  /*2字节，地址家族*/
    unsigned short int sin_port;  /*2字节，端口号*/
    struct in_addr sin_addr; /*4字节，IP地址*/
    unsigned char sin_zero[8]; /*8字节，对齐*/
};
```
### 1.1.3. in_addr
```c
typedef struct in_addr {
    union {
        struct{ unsigned char s_b1,s_b2, s_b3,s_b4;} S_un_b;
        struct{ unsigned short s_w1, s_w2;} S_un_w;
        unsigned long S_addr;
    } S_un;
} IN_ADDR;
```
## 1.2. 字符串地址与in_addr地址的相互转换
```c
addrto.sin_addr.s_addr = inet_addr("192.168.0.2");
str = inet_ntoa(addrto.sin_addr);
```

