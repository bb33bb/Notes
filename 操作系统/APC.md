<!-- TOC -->

- [1. APC的作用：改变线程行为](#1-apc的作用改变线程行为)
- [2. _KTHREAD结构体中APC相关成员](#2-_kthread结构体中apc相关成员)
    - [2.1. APC队列：ApcState](#21-apc队列apcstate)
    - [2.2. 备用APC队列：SavedApcState](#22-备用apc队列savedapcstate)
    - [2.3. APC寻址](#23-apc寻址)
- [3. APC的挂入过程](#3-apc的挂入过程)
    - [3.1. KAPC](#31-kapc)
    - [3.2. 挂入流程](#32-挂入流程)
    - [3.3. KeInitializeAPC](#33-keinitializeapc)
        - [3.3.1. 函数定义](#331-函数定义)
        - [3.3.2. ApcStateIndex](#332-apcstateindex)
    - [3.4. KeInsertQueueAPC](#34-keinsertqueueapc)
- [4. APC的执行时机](#4-apc的执行时机)

<!-- /TOC -->
# 1. APC的作用：改变线程行为
线程在运行的时候是不受其它线程控制的，不会被结束、挂起、恢复。如果想要改变线程行为，可以通过给线程提供一个函数，让线程自己去调用的形式来实现。这个函数就是APC（Asyncroneus Procedure Call，异步过程调用）
# 2. _KTHREAD结构体中APC相关成员
```x86asm
kd> dt _kthread
nt!_KTHREAD
......
   +0x034 ApcState         : _KAPC_STATE    ;APC队列
......
   +0x138 ApcStatePointer  : [2] Ptr32 _KAPC_STATE    ;正常情况下，ApcStatePointer[0]指向ApcState，ApcStatePointer[1]指向SavedApcState，线程挂靠情况下，指针指向相反
......
   +0x14c SavedApcState    : _KAPC_STATE    ;备用APC队列
......
   +0x165 ApcStateIndex    : UChar          ;标识当前线程处于什么状态，0表示正常状态，1表示挂靠状态
   +0x166 ApcQueueable     : UChar          ;表示是否可以向线程的APC队列中插入APC，当线程正在执行退出的代码时，会将这个值设置为0，如果此时执行插入APC的代码（KeInsertQueueApc），在插入函数中会判断这个值是否为0，如果是则插入失败
......
```
## 2.1. APC队列：ApcState
```x86asm
kd> dt _KAPC_STATE
nt!_KAPC_STATE
   +0x000 ApcListHead      : [2] _LIST_ENTRY   ;APC队列，两个双向链表，里面存储了线程需要执行的APC函数。用户APC链表（函数为用户空间函数）和内核APC链表（函数为内核空间函数）
   +0x010 Process          : Ptr32 _KPROCESS   ;指向为线程提供CR3的进程
   +0x014 KernelApcInProgress : UChar          ;指示内核APC函数是否正在执行
   +0x015 KernelApcPending : UChar             ;是否存在内核APC函数，存在则置1，不存在置0
   +0x016 UserApcPending   : UChar             ;是否存在用户APC函数，存在则置1，不存在置0
```
## 2.2. 备用APC队列：SavedApcState
对于A进程中的线程，其APC函数所使用的内存空间是A进程的内存空间，但是当线程挂靠到B进程之后，CR3切换导致内存空间切换，APC函数中使用的地址将会出现错误。所以在发生进程挂靠之后，会将ApcState中的内容存储到SavedApcState（备用APC队列）中。这时线程的所属进程为B进程，这时候再往线程中插入APC，插入的是ApcState（ApcState.Process也是指向B进程）。等线程回到A进程之后，会再将SavedApcState中内容恢复到ApcState中。
## 2.3. APC寻址
Windows使用ApcStatePointer和ApcStateIndex联合进行APC寻址，无论线程处于正常情况下还是挂靠情况下，ApcStatePointer[ApcStateIndex]均指向ApcState，ApcState则总是表示线程当前使用的APC状态
# 3. APC的挂入过程
每当要挂入一个APC的时候，无论是内核APC还是用户APC，内核都要准备一个KAPC的数据结构，并且将这个KAPC结构挂入相应的APC队列。
## 3.1. KAPC
```
kd> dt _KAPC
nt!_KAPC
   +0x000 Type             : Int2B           ;Windows内核对象（进程、线程、事件等）类型，APC为0x12
   +0x002 Size             : Int2B           ;本结构体大小，0x30
   +0x004 Spare0           : Uint4B          ;未使用
   +0x008 Thread           : Ptr32 _KTHREAD  ;目标线程
   +0x00c ApcListEntry     : _LIST_ENTRY     ;APC队列所挂位置
   +0x014 KernelRoutine    : Ptr32     void  ;指向一个函数，APC函数执行完毕之后Windows会调用这个函数，该函数需调用ExFreePoolWithTag来销毁KAPC结构体 
   +0x018 RundownRoutine   : Ptr32     void  ;未使用
   +0x01c NormalRoutine    : Ptr32     void  ;用户APC：用户APC总入口；内核APC：函数地址
   +0x020 NormalContext    : Ptr32 Void      ;内核APC：NULL；用户APC：函数地址
   +0x024 SystemArgument1  : Ptr32 Void      ;APC参数1
   +0x028 SystemArgument2  : Ptr32 Void      ;APC参数2
   +0x02c ApcStateIndex    : Char            ;标识希望APC挂在哪个队列
   +0x02d ApcMode          : Char            ;0代表内核APC，1代表用户APC
   +0x02e Inserted         : UChar           ;标识当前APC是否已经插入队列，是为1，否为0
```
## 3.2. 挂入流程
QueueUserAPC(位于Kernel32.dll)，该函数会调用NtQueueApcThread(位于ntoskrnl.exe)函数，该函数会分配KAPC的空间，然后调用KeInitializeAPC(位于ntoskrnl.exe)来初始化KAPC结构体，然后调用KeInsertQueueApc，该函数会调用KiInsertQueueApc将KAPC插入指定队列。三环插入APC的时候调用QueueUserAPC，而内核插入APC的时候可能会直接调用KeInitializeAPC和KiInsertQueueApc来插入APC。
## 3.3. KeInitializeAPC
### 3.3.1. 函数定义
```c
VOID KeInitializeApc ( 
   IN PKAPC Apc,    //KAPC指针
   IN PKTHREAD Thread, //目标线程
   IN KAPC_ENVIRONMENT TargetEnvironment,   //标识希望APC挂在哪个队列，与KAPC.ApcStateIndex相对应
   IN PKKERNEL_ROUTINE KernelRoutine,       //销毁KAPC的函数地址
   IN PKRUNDOWN_ROUTINE RundownRoutine OPTIONAL,      //未使用
   IN PKNORMAL_ROUTINE NormalRoutine,    //用户APC：用户APC总入口；内核APC：函数地址
   IN KPROCESSOR_MODE Mode,      //0代表内核APC，1代表用户APC
   IN PVOID Context     //用户APC：函数地址；内核APC：NULL
);
```
### 3.3.2. ApcStateIndex
与KTHREAD + 0x165的成员同名，但是含义不同。ApcStateIndex有以下4个值：
* 0：原始环境，即希望APC挂到原始线程的APC队列，ApcStatePointer[0]
* 1：挂靠环境，即希望APC挂到挂靠线程的APC队列，ApcStatePointer[1]
* 2：初始化APC时的当前环境，即希望APC挂到当前线程（取初始化时的当前线程）的APC队列，ApcStatePointer[KTHREAD.ApcStateIndex]
* 3：插入APC时的当前环境，即希望APC挂到当前线程（取插入时的当前线程，从初始化到插入的过程中，可能发生挂靠或者接触挂靠）的APC队列,ApcStatePointer[KTHREAD.ApcStateIndex]
## 3.4. KeInsertQueueAPC
* 根据KAPC结构中的ApcStateIndex找到对应的APC队列
* 根据KAPC结构中的ApcMode确定是内核APC还是用户APC
* 将KAPC挂到相应队列（挂到KAPC的ApcListEntry处）
* 将KAPC结构中的Inserted置1，标识当前APC已经插入
* 修正KAPC_STATE结构中的KernelApcPending和UserApcPending
# 4. APC的执行时机
KiServiceExit函数是系统调用、异常或者中断返回用户空间的必经之路，在这个函数中，会判断是否存在用户APC，之后调用KiDeliverApc函数执行内核APC函数，之后执行用户APC函数（如果存在）