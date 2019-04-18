<!-- TOC -->

- [1. ELF文件](#1-elf文件)
    - [1.1. ELF文件格式](#11-elf文件格式)
    - [1.2. ELF文件分析常用工具](#12-elf文件分析常用工具)
- [2. 反调试技术](#2-反调试技术)
    - [2.1. ptrace自身进程](#21-ptrace自身进程)
    - [2.2. 检查父进程名称](#22-检查父进程名称)
    - [2.3. 检查进程运行状态](#23-检查进程运行状态)
    - [2.4. 设置程序运行最大时间](#24-设置程序运行最大时间)
    - [2.5. 检查进程打开的filedescriptor](#25-检查进程打开的filedescriptor)

<!-- /TOC -->
# 1. ELF文件
ELF(Executable and Linkable Format)是Unix及类Unix系统下可执行文件、共享库等二进制文件标准格式。
## 1.1. ELF文件格式
ELF文件主要由以下几部分组成：
* ELF header
* Program header table，对应于segments
* Section header table，对应于sections
* 被Program header table或Sectionheader table指向的内容

>**注意**：segments部分包含各segments的地址、偏移、属性等；而sections部分则依次列出每个segment所包含的sections。ections通常为全小写字母，segments通常为全大写字母。

注意这里segments与sections是两个不同的概念。之间的关系如下：
* 每个segments可以包含多个sections
* 每个sections可以属于多个segments
* segments之间可以有重合的部分
* sections包含的是链接时需要的信息，链接器通过section header table去寻找sections
* segments包含运行时需要的信息，加载器通过program header table去寻找segments
## 1.2. ELF文件分析常用工具
* ptrace，系统调用。用于监控其他进程，被gdb, strace, ltrace等使用
* strace，命令行工具。用于追踪进程与内核的交互，如系统调用、信号传递
* ltrace，命令行工具。类似于strace，但主要用于追踪库函数调用
* readelf/objdump，静态分析工具。用于读取ELF文件信息。
# 2. 反调试技术
## 2.1. ptrace自身进程
在同一时间，进程最多只能被一个调试器进行调试。于是，我们可以通过调试进程自身，来判断是否已经有其他进程（调试器）的存在。
```c
#include <stdio.h>
#include <sys/ptrace.h>
int main(int argc, char *argv[]) {
    if(ptrace(PTRACE_TRACEME, 0, 0, 0) == -1) {
       printf("Debugger detected");
       return 1;
   }  
   printf("All good");
   return 0;
}
```
## 2.2. 检查父进程名称
通常，我们在使用gdb调试时，是通过gdb <TARGET>这种方式进行的。而这种方式是启动gdb，fork出子进程后执行目标二进制文件。因此，二进制文件的父进程即为调试器。
```c
#include <stdio.h>
#include <string.h>
 
int main(int argc, char *argv[]) {
   char buf0[32], buf1[128];
   FILE* fin;
 
   snprintf(buf0, 24, "/proc/%d/cmdline", getppid());
   fin = fopen(buf0, "r");
   fgets(buf1, 128, fin);
   fclose(fin);
 
   if(!strcmp(buf1, "gdb")) {
       printf("Debugger detected");
       return 1;
   }  
   printf("All good");
   return 0;
}
```
## 2.3. 检查进程运行状态
使用调试器附加到进程之后，/proc/self/status文件内容会发生变化，进程状态由sleeping变为tracing stop，TracerPid也由0变为非0的数，即调试器的PID。
```c
#include <stdio.h>
#include <string.h>
int main(int argc, char *argv[]) {
   int i;
   scanf("%d", &i);
   char buf1[512];
   FILE* fin;
   fin = fopen("/proc/self/status", "r");
   int tpid;
   const char *needle = "TracerPid:";
   size_t nl = strlen(needle);
   while(fgets(buf1, 512, fin)) {
       if(!strncmp(buf1, needle, nl)) {
           sscanf(buf1, "TracerPid: %d", &tpid);
           if(tpid != 0) {
                printf("Debuggerdetected");
                return 1;
           }
       }
    }
   fclose(fin);
   printf("All good");
   return 0;
}
```
## 2.4. 设置程序运行最大时间
调试过程很耗费时间，运行时间要远远大于正常运行时间，可以在程序启动时，通过alarm设置定时，到达时则中止程序。
```c
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
void alarmHandler(int sig) {
   printf("Debugger detected");
   exit(1);
}
void__attribute__((constructor))setupSig(void) {
   signal(SIGALRM, alarmHandler);
   alarm(2);
}
int main(int argc, char *argv[]) {
   printf("All good");
   return 0;
}
```
如果GDB选择忽略signal，这个方法不会奏效
## 2.5. 检查进程打开的filedescriptor
如果被调试的进程是通过gdb <TARGET>的方式启动，那么它便是由gdb进程fork得到的。而fork在调用时，父进程所拥有的fd(file descriptor)会被子进程继承。由于gdb在往往会打开多个fd，因此如果进程拥有的fd较多，则可能是继承自gdb的，即进程在被调试。 
```c
#include <stdio.h>
#include <string.h>
 
int main(int argc, char *argv[]) {
   char buf0[32], buf1[128];
   FILE* fin;
 
   snprintf(buf0, 24, "/proc/%d/cmdline", getppid());
   fin = fopen(buf0, "r");
   fgets(buf1, 128, fin);
   fclose(fin);
 
   if(!strcmp(buf1, "gdb")) {
       printf("Debugger detected");
       return 1;
   }  
   printf("All good");
   return 0;
}
```