<!-- TOC -->

- [1. APC的作用：改变线程行为](#1-apc的作用改变线程行为)
- [2. _KTHREAD结构体中APC相关成员](#2-_kthread结构体中apc相关成员)
    - [2.1. APC队列：ApcState](#21-apc队列apcstate)
    - [2.2. 备用APC队列：SavedApcState](#22-备用apc队列savedapcstate)
    - [2.3. APC寻址](#23-apc寻址)
- [3. APC的挂入过程](#3-apc的挂入过程)
    - [3.1. 挂入流程](#31-挂入流程)
    - [3.2. KeInitializeAPC](#32-keinitializeapc)
        - [3.2.1. 函数定义](#321-函数定义)
        - [3.2.2. TargetEnvironment](#322-targetenvironment)
    - [3.3. KeInsertQueueAPC](#33-keinsertqueueapc)
- [4. APC的执行时机](#4-apc的执行时机)

<!-- /TOC -->
# 1. APC的作用：改变线程行为
线程在运行的时候是不受其它线程控制的，不会被结束、挂起、恢复。如果想要改变线程行为，可以通过给线程提供一个函数，让线程自己去调用的形式来实现。这个函数就是APC（Asyncroneus Procedure Call，异步过程调用）
# 2. _KTHREAD结构体中APC相关成员
## 2.1. APC队列：ApcState
ApcState为线程的Apc队列，类型为_KAPC_STATE（结构见《内核结构体详解》）。
## 2.2. 备用APC队列：SavedApcState
对于A进程中的线程，其APC函数所使用的内存空间是A进程的内存空间，但是当线程挂靠到B进程之后，CR3切换导致内存空间切换，APC函数中使用的地址将会出现错误。所以在发生进程挂靠之后，会将ApcState中的内容存储到SavedApcState（备用APC队列）中。这时线程的所属进程为B进程，这时候再往线程中插入APC，插入的是ApcState（ApcState.Process也是指向B进程）。等线程回到A进程之后，会再将SavedApcState中内容恢复到ApcState中。
## 2.3. APC寻址
Windows使用ApcStatePointer和ApcStateIndex联合进行APC寻址，无论线程处于正常情况下还是挂靠情况下，ApcStatePointer[ApcStateIndex]均指向ApcState，ApcState则总是表示线程当前使用的APC状态
# 3. APC的挂入过程
每当要挂入一个APC的时候，无论是内核APC还是用户APC，内核都要准备一个_KAPC的数据结构（结构见《内核结构体详解》），并且将这个KAPC结构挂入相应的APC队列。
## 3.1. 挂入流程
QueueUserAPC(位于Kernel32.dll)，该函数会调用NtQueueApcThread(位于ntoskrnl.exe)函数，该函数会分配KAPC的空间，然后调用KeInitializeAPC(位于ntoskrnl.exe)来初始化KAPC结构体，然后调用KeInsertQueueApc，该函数会调用KiInsertQueueApc将KAPC插入指定队列。三环插入APC的时候调用QueueUserAPC，而内核插入APC的时候可能会直接调用KeInitializeAPC和KiInsertQueueApc来插入APC。
## 3.2. KeInitializeAPC
### 3.2.1. 函数定义
```c
VOID KeInitializeApc ( 
    IN PKAPC Apc,    //KAPC指针
    IN PKTHREAD Thread, //目标线程
    IN KAPC_ENVIRONMENT TargetEnvironment,   //标识希望APC挂在哪个队列
    IN PKKERNEL_ROUTINE KernelRoutine,       //销毁KAPC的函数地址
    IN PKRUNDOWN_ROUTINE RundownRoutine OPTIONAL,      //未使用
    IN PKNORMAL_ROUTINE NormalRoutine,    //用户APC：用户APC总入口；内核APC：函数地址
    IN KPROCESSOR_MODE Mode,      //0代表内核APC，1代表用户APC
    IN PVOID Context     //用户APC：函数地址；内核APC：NULL
);
```
### 3.2.2. TargetEnvironment
* 0：原始环境，即希望APC挂到原始线程的APC队列，ApcStatePointer[0]
* 1：挂靠环境，即希望APC挂到挂靠线程的APC队列，ApcStatePointer[1]
* 2：初始化APC时的当前环境，即希望APC挂到当前线程（取初始化时的当前线程即在KeInitializeApc中获取）的APC队列，ApcStatePointer[KTHREAD.ApcStateIndex]
* 3：插入APC时的当前环境，即希望APC挂到当前线程（取插入时的当前线程即在KeInsertQueueAPC -> KiInsertQueueAPC中获取，从初始化到插入的过程中，可能发生挂靠或者接触挂靠）的APC队列,ApcStatePointer[KTHREAD.ApcStateIndex]
## 3.3. KeInsertQueueAPC
* 根据KAPC结构中的ApcStateIndex找到对应的APC队列
* 根据KAPC结构中的ApcMode确定是内核APC还是用户APC
* 将KAPC挂到相应队列（挂到KAPC的ApcListEntry处）
* 将KAPC结构中的Inserted置1，标识当前APC已经插入
* 修正KAPC_STATE结构中的KernelApcPending和UserApcPending。如果APC为内核APC，将KernelApcPending置1。如果APC为用户APC且目标线程位于等待状态、而且是用户（如用户程序自己Sleep、WaitForSingleObject）导致的等待（非内核程序让它等待）、而且可以被唤醒，将UserApcPending置1并唤醒目标线程（从等待链表中摘除，放到调度链表），其它情况下UserApcPending不修正。如果UserApcPending本来为0，又未得到修正，且之后没有其它用户APC的插入导致UserApcPending被置1，该APC会无法得到执行时机
# 4. APC的执行时机
KiServiceExit函数是系统调用、异常或者中断返回用户空间的必经之路，在这个函数中，会通过UserApcPending判断是否存在用户APC，之后调用KiDeliverApc函数执行内核APC函数，之后执行用户APC函数（如果存在）