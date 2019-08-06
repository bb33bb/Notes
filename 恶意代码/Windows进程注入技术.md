<!-- TOC -->

- [1. 概念](#1-概念)
- [2. DLL注入](#2-dll注入)
    - [2.1. 经典DLL注入](#21-经典dll注入)
        - [2.1.1. 局限性](#211-局限性)
    - [2.2. 反射DLL注入（ ReflectiveDLL Injection）](#22-反射dll注入-reflectivedll-injection)
    - [2.3. DLL感染及替换](#23-dll感染及替换)
    - [2.4. DLL加载顺序劫持](#24-dll加载顺序劫持)
    - [2.5. 利用输入法注入DLL](#25-利用输入法注入dll)
- [3. PORTABLE EXECUTABLE注入（PE注入、代码注入）](#3-portable-executable注入pe注入代码注入)
    - [3.1. 局限性](#31-局限性)
- [4. PROCESS HOLLOWING技术（进程替换）](#4-process-hollowing技术进程替换)
- [5. 线程执行劫持技术](#5-线程执行劫持技术)
- [6. 通过SETWINDOWSHOOKEX进行HOOK注入](#6-通过setwindowshookex进行hook注入)
- [7. 通过注册表实现注入和持久性](#7-通过注册表实现注入和持久性)
    - [7.1. AppInit_DLLs](#71-appinit_dlls)
    - [7.2. AppCertDlls](#72-appcertdlls)
    - [7.3. 映像文件执行选项（IFEO）](#73-映像文件执行选项ifeo)
- [8. APC注入和ATOMBOMBING](#8-apc注入和atombombing)
    - [8.1. 内核模式APC（为系统和驱动生成）](#81-内核模式apc为系统和驱动生成)
    - [8.2. 用户模式APC（为应用程序生成）](#82-用户模式apc为应用程序生成)
    - [8.3. AtomBombing](#83-atombombing)
- [9. 通过SETWINDOWLONG进行附加窗口内存注入（EWMI）](#9-通过setwindowlong进行附加窗口内存注入ewmi)
- [10. SHIMS注入](#10-shims注入)
- [11. Hook技术](#11-hook技术)

<!-- /TOC -->
# 1. 概念
进程注入是一种广泛使用的躲避检测的技术，通常用于恶意软件或者无文件技术。其需要在另一个进程的地址空间内运行特制代码，进程注入改善了不可见性，同时一些技术也实现了持久性。
# 2. DLL注入
## 2.1. 经典DLL注入
* 提权，获取SeDebugPrivilege权限，保证自己可以访问别的进程的内存空间
* 通过CreateToolhelp32Snapshot，Process32First和Process32Next找到目标进程
* 调用OpenProcess获取目标进程的句柄，调用VirtualAllocEx和WriteProcessMemory分配空间并写入DLL文件路径
* 通过CreateRemoteThread，NtCreateThreadEx或RtlCreateUserThread调用LoadLibrary函数，强制其加载恶意DLL
### 2.1.1. 局限性
* 需要dll文件落地磁盘，易被查杀
* CreateRemoteThread为杀毒软件重点监控函数
## 2.2. 反射DLL注入（ ReflectiveDLL Injection）
经典DLL注入由于太过套路化，而且需要DLL文件落地磁盘，所以容易被杀毒软件检测。反射DLL注入需要一个注射器和待注入DLL。注射器可以通过网络、提取自身节、读取加密文件等方式来避免DLL文件落地磁盘，之后，注射器将待注入DLL写入目标进程内存空间内，并创建远程线程调用DLL的导出函数ReflectiveLoader（无法使用全局变量，需要满足PIC即地址无关代码）来对DLL完成手动加载操作，手动加载操作步骤如下：
* 获取基地址（PEB、call-pop并向上搜索“MZ”）
* 获取Kernel32基地址（PEB），获取LoadLibraryA、GetProcAddress、VirtualAlloc、NtFlushInstructionCache四个函数的地址
* 分配空间并将DLL展开映射（文件->内存的过程）
* 处理DLL的导入表
* 处理重定位
* 处理TLS
* 调用NtFlushInstructionCache刷新指令缓存
* 调用DLLMain
## 2.3. DLL感染及替换
对于进程需要使用到的dll、系统dll进行替换或者感染
## 2.4. DLL加载顺序劫持
## 2.5. 利用输入法注入DLL
# 3. PORTABLE EXECUTABLE注入（PE注入、代码注入）
这种技术类似于经典DLL注入，但是在分配空间后，它直接写入恶意代码并执行（通过shellcode或调用CreateRemoteThread），所以不需要在磁盘上存放恶意DLL文件。但是这种技术需要进行手动的地址重定位。
## 3.1. 局限性
* 函数体直接使用API时不得使用不属于kernel32.dll、user32.dll的API，因为只有这两个模块在各个进程里的基地址是一样的
* 函数体不能使用任何字符串常量，因为字符串常量存放在.data段里，不会随着代码注入目标进程
* 编译器不得使用GZ编译选项，因为这个选项会在函数体中加入一些代码，用来检验ESP在函数体中是否被改变，但是这些检验函数的地址在不同进程中可能不一样
* 不得使用增量链接，增量链接会把函数体用一个JMP指令代替，这样就可以改变函数的内容而不用修改CALL指令
* 不得在函数体内使用超过4kb的局部变量，当定义的局部变量的大小超过4kb时，不会直接修改栈指针，而是调用另一个函数来分配内存，这个函数在不同进程中的地址可能不一样
* 函数体内switch语句中的case不要超过3个，否则编译器会使用跳转表，而这个跳转表并不位于函数体中
# 4. PROCESS HOLLOWING技术（进程替换）
恶意软件首先会创建一个新进程并挂起，之后，恶意软件通过调用ZwUnmapViewOfSection或NtUnmapViewOfSection来取消映射目标进程的内存，即卸下目标进程中的代码，之后执行VirtualAllocEx分配新内存，并使用WriteProcessMemory将恶意代码写入目标进程空间，之后利用SetThreadContext将入口点指向恶意代码。最后，恶意软件通过调用ResumeThread恢复挂起的线程，使进程退出挂起状态。
# 5. 线程执行劫持技术
这种技术类似与Process Hollowing，它以以进程的现有线程为目标。在通过CreateToolhelp32Snapshot和OpenThread获取目标线程的句柄后，恶意软件通过调用SuspandThread来挂起这个线程，然后调用VirtualAllocEx和WriteProcessMemory来分配内存并执行代码注入。代码可以包含shellcode，恶意DLL的路径以及LoadLibrary的地址。 为了劫持线程的执行，恶意软件通过调用SetThreadContext来修改目标线程的EIP寄存器（包含下一条指令的地址的寄存器）。之后，恶意软件恢复线程来执行它已写入主机进程的shellcode。由于在系统调用过程中挂起和恢复线程会导致系统崩溃，如果EIP寄存器在NTDLL.dll范围内，恶意软件会在稍后重新尝试注入。
# 6. 通过SETWINDOWSHOOKEX进行HOOK注入
SetWindowsHookEx函数有四个参数。第一个参数是事件的类型。事件反映了HOOK类型的范围，从键盘上的按键（WH_KEYBOARD）到鼠标输入（WH_MOUSE），CBT等等。第二个参数是指向恶意软件想要在事件上调用的函数的指针。第三个参数是包含该函数的模块。因此，在调用SetWindowsHookEx之前，通常会看到对LoadLibrary和GetProcAddress的调用。此函数的最后一个参数是与HOOK过程相关联的线程。如果此值设置为零，则所有线程都会在触发事件时执行操作。但是，恶意软件通常针对一个线程以降低噪声，因此在SetWindowsHookEx之前也可以看到调用CreateToolhelp32Snapshot和Thread32Next来查找和定位单个线程。注入DLL后，恶意软件代表其threadId传递给SetWindowsHookEx函数的进程执行其恶意代码。
# 7. 通过注册表实现注入和持久性
## 7.1. AppInit_DLLs
* HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows\Appinit_Dlls
* HKLM\Software\Wow6432Node\Microsoft\Windows NT\CurrentVersion\Windows\Appinit_Dlls

此注册表项下的每个库都会加载到每个加载User32.dll的进程中。
## 7.2. AppCertDlls
* HKLM\System\CurrentControlSet\Control\Session Manager\AppCertDlls

只是此注册表项下的DLL被加载到调用Win32 API函数CreateProcess，CreateProcessAsUser，CreateProcessWithLogonW，CreateProcessWithTokenW和WinExec的每个进程中。
## 7.3. 映像文件执行选项（IFEO）
* HKLM\Software\Microsoft\Windows NT\currentversion\image file execution options

开发人员可以在此注册表项下设置“调试器值”，每当启动可执行文件时，会启动调试器并附加。要使用此功能，你只需提供调试器的路径，并将其附加到要分析的可执行文件。
# 8. APC注入和ATOMBOMBING 
恶意软件可以利用异步过程调用（APC）通过将其附加到目标线程的APC队列来强制另一个线程执行其特制代码。每个线程都有一个APC队列，它们等待目标线程进入可变状态时执行。如果线程调用SleepEx，SignalObjectAndWait，MsgWaitForMultipleObjectsEx，WaitForMultipleObjectsEx或WaitForSingleObjectEx函数，则线程进入可更改状态。恶意软件通常会查找处于可更改状态的任何线程，然后调用OpenThread和QueueUserAPC将APC排入线程。 QueueUserAPC有三个参数：
* 目标线程的句柄;
* 指向恶意软件想要运行的功能的指针
* 传递给函数指针的参数
## 8.1. 内核模式APC（为系统和驱动生成）
## 8.2. 用户模式APC（为应用程序生成）
## 8.3. AtomBombing
Amanahe恶意软件首先调用OpenThread来获取另一个线程的句柄，然后通过LoadLibraryA调用QueueUserAPC作为函数指针，将其恶意DLL注入另一个线程。AtomBombing是一项由enSilo研究首次引入的技术，然后用于Dridex V4。 正如我们在前一篇文章中详细讨论的那样，该技术也依赖于APC注入。 但是，它使用原子表写入另一个进程的内存。
# 9. 通过SETWINDOWLONG进行附加窗口内存注入（EWMI）
EWMI依赖于注入资源管理器托盘窗口的额外窗口内存，并且已经在Gapz和PowerLoader等恶意软件系列中应用过几次。注册窗口类时，应用程序可以指定一些额外的内存字节，称为额外窗口内存（EWM）。但是，EWM的空间不大。为了规避此限制，恶意软件将代码写入explorer.exe的共享部分，并使用SetWindowLong和SendNotifyMessage使用指向shellcode的函数指针，然后执行它。

在写入共享部分时，恶意软件有两种选择。它既可以创建共享空间，也可以将其映射到自身和另一个进程（例如explorer.exe），也可以只打开已存在的共享空间。除了一些其他API调用之外，前者还有分配堆空间和调用NTMapViewOfSection的开销，因此后一种方法更常用。在恶意软件将其shellcode写入共享部分后，它使用GetWindowLong和SetWindowLong来访问和修改“Shell_TrayWnd”的额外窗口内存。GetWindowLong是一个API，用于将指定偏移量的32位值检索到窗口类对象的额外窗口内存中，SetWindowLong用于更改指定偏移量的值。这样一来，恶意软件可以简单地更改窗口类中的函数指针的偏移量，并将其指向写入共享部分的shellcode。

使用此方法，恶意软件会通过调用SendNotifyMessage来触发注入的代码。执行SendNotifyMessage后，Shell_TrayWnd接收控制并将控制转移到之前由SetWindowLong设置的值指向的地址。
# 10. SHIMS注入
SHIMS允许开发人员将修补程序应用于他们的程序，而无需重写代码，主要是为了向后兼容。SHIMS本质上是一种挂钩API并定位特定可执行文件的方法。恶意软件可以利用SHIMS来定位持久性和注入的可执行文件。Windows在加载二进制文件时运行Shim Engine以检查SHIMS数据库以应用适当的修复程序。
# 11. Hook技术
[Hook技术](../操作系统/Hook技术.md)