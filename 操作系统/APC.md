<!-- TOC -->

- [1. APC的作用：改变线程行为](#1-apc的作用改变线程行为)
- [2. APC的结构](#2-apc的结构)
- [3. _KTHREAD结构体中APC相关成员](#3-_kthread结构体中apc相关成员)
- [4. APC队列：ApcState](#4-apc队列apcstate)
- [5. 备用APC队列：SavedApcState](#5-备用apc队列savedapcstate)
- [6. APC寻址](#6-apc寻址)
- [7. APC的执行时机](#7-apc的执行时机)

<!-- /TOC -->
# 1. APC的作用：改变线程行为
线程在运行的时候是不受其它线程控制的，不会被结束、挂起、恢复。如果想要改变线程行为，可以通过给线程提供一个函数，让线程自己去调用的形式来实现。这个函数就是APC（Asyncroneus Procedure Call，异步过程调用）
# 2. APC的结构
```
kd> dt _KAPC
nt!_KAPC
   +0x000 Type             : Int2B
   +0x002 Size             : Int2B
   +0x004 Spare0           : Uint4B
   +0x008 Thread           : Ptr32 _KTHREAD
   +0x00c ApcListEntry     : _LIST_ENTRY
   +0x014 KernelRoutine    : Ptr32     void 
   +0x018 RundownRoutine   : Ptr32     void
   +0x01c NormalRoutine    : Ptr32     void  ;找到APC函数，但是这个指针并不一定指向函数地址
   +0x020 NormalContext    : Ptr32 Void
   +0x024 SystemArgument1  : Ptr32 Void
   +0x028 SystemArgument2  : Ptr32 Void
   +0x02c ApcStateIndex    : Char
   +0x02d ApcMode          : Char
   +0x02e Inserted         : UChar
```
# 3. _KTHREAD结构体中APC相关成员
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
# 4. APC队列：ApcState
```x86asm
kd> dt _KAPC_STATE
nt!_KAPC_STATE
   +0x000 ApcListHead      : [2] _LIST_ENTRY   ;APC队列，两个双向链表，里面存储了线程需要执行的APC函数。用户APC链表（函数为用户空间函数）和内核APC链表（函数为内核空间函数）
   +0x010 Process          : Ptr32 _KPROCESS   ;指向为线程提供CR3的进程
   +0x014 KernelApcInProgress : UChar          ;指示内核APC函数是否正在执行
   +0x015 KernelApcPending : UChar             ;是否存在内核APC函数，存在则置1，不存在置0
   +0x016 UserApcPending   : UChar             ;是否存在用户APC函数，存在则置1，不存在置0
```
# 5. 备用APC队列：SavedApcState
对于A进程中的线程，其APC函数所使用的内存空间是A进程的内存空间，但是当线程挂靠到B进程之后，CR3切换导致内存空间切换，APC函数中使用的地址将会出现错误。所以在发生进程挂靠之后，会将ApcState中的内容存储到SavedApcState（备用APC队列）中。这时线程的所属进程为B进程，这时候再往线程中插入APC，插入的是ApcState（ApcState.Process也是指向B进程）。等线程回到A进程之后，会再将SavedApcState中内容恢复到ApcState中。
# 6. APC寻址
Windows使用ApcStatePointer和ApcStateIndex联合进行APC寻址，无论线程处于正常情况下还是挂靠情况下，ApcStatePointer[ApcStateIndex]均指向ApcState，ApcState则总是表示线程当前使用的APC状态
# 7. APC的执行时机
KiServiceExit函数是系统调用、异常或者中断返回用户空间的必经之路，在这个函数中，会判断是否存在用户APC，之后调用KiDeliverApc函数执行内核APC函数，之后执行用户APC函数（如果存在）