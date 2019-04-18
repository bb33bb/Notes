<!-- TOC -->

- [1. Hook分类](#1-hook分类)
    - [1.1. 按照挂钩对象分类](#11-按照挂钩对象分类)
    - [1.2. 按照挂钩数量分类](#12-按照挂钩数量分类)
- [2. Ring3层Hook](#2-ring3层hook)
    - [2.1. 系统消息Hook](#21-系统消息hook)
    - [2.2. lpk Hook](#22-lpk-hook)
    - [2.3. Inline Hook（API Hook）](#23-inline-hookapi-hook)
        - [2.3.1. 步骤](#231-步骤)
        - [2.3.2. 注意事项](#232-注意事项)
    - [2.4. IAT Hook](#24-iat-hook)
    - [2.5. SEH Hook](#25-seh-hook)
    - [2.6. 实现全局Hook](#26-实现全局hook)
- [3. Ring0层Hook](#3-ring0层hook)
    - [3.1. IRP Hook](#31-irp-hook)
    - [3.2. MSR Hook](#32-msr-hook)
    - [3.3. SSDT Hook](#33-ssdt-hook)
- [4. Hook方式](#4-hook方式)
    - [4.1. 手动Hook](#41-手动hook)
    - [4.2. Hook库](#42-hook库)
- [5. Hook注意事项](#5-hook注意事项)
    - [5.1. _declspec(naked)](#51-_declspecnaked)

<!-- /TOC -->
# 1. Hook分类
## 1.1. 按照挂钩对象分类
* 本地钩子：挂钩本进程。
* 远程钩子：挂钩其他进程。
    * 上层钩子：钩子位于对方进程。
    * 底层钩子：钩子位于安装钩子的进程，涉及到跨进程通信。
## 1.2. 按照挂钩数量分类
* 只挂钩一个进程
* 挂钩多个进程
* 挂钩全部进程（对于新创建的进程也要进行挂钩）

# 2. Ring3层Hook
## 2.1. 系统消息Hook
通过微软官方提供的API进行消息Hook。
* SetWindowsHookEx
* UnhookWindowsHookEx
* CallNextHookEx

进行系统消息Hook时，可以通过向目标进程发送消息的方法来强制目标进程触发挂钩
## 2.2. lpk Hook
利用Windows提供的未公开函数InitializeLpkHooks可以HOOK位于Windows中lpk.dll（用于支持多语言包功能）的4个函数LpkTabbedTextOut，LpkPSMTextOut，LpkDrawTextEx，LpkEditControl。
```vb
Private Sub Form_Load() 
    DLLhwnd = LoadLibrary("lpk.dll") '加载 DLL 
    DLLFunDre = GetProcAddress(DLLhwnd, "LpkDrawTextEx") '获取回调函数地址

    LpkHooksInfo.lpHookProc_LpkTabbedTextOut = 0 
    LpkHooksInfo.lpHookProc_LpkPSMTextOut = 0 
    LpkHooksInfo.lpHookProc_LpkDrawTextEx = GetLocalProcAdress(AddressOf HookProc1) '设置要 HOOK 的 LPK 函数
    LpkHooksInfo.lpHookProc_LpkEditControl = 0 
    InitializeLpkHooks LpkHooksInfo 
End Sub 
Private Sub Form_Unload(Cancel As Integer) 
    LpkHooksInfo.lpHookProc_LpkTabbedTextOut = 0 
    LpkHooksInfo.lpHookProc_LpkPSMTextOut = 0 
    LpkHooksInfo.lpHookProc_LpkDrawTextEx = DLLFunDre 
    LpkHooksInfo.lpHookProc_LpkEditControl = 0 
    InitializeLpkHooks LpkHooksInfo 
    FreeLibrary DLLhwnd 
End Sub 
```
然后新建一个模块，在模块中加入以下代码
```vb
Public Declare Function LoadLibrary Lib "kernel32" Alias "LoadLibraryA" (ByVal lpLibFileName As String) As Long 
Public Declare Function GetProcAddress Lib "kernel32" (ByVal hModule As Long, ByVal lpProcName As String) As Long 
Public Declare Function FreeLibrary Lib "kernel32" (ByVal hLibModule As Long) As Long
Public Declare Sub InitializeLpkHooks Lib "user32" (lpProcType As Any) 

Type LpkHooksSetting 
    lpHookProc_LpkTabbedTextOut As Long 
    lpHookProc_LpkPSMTextOut As Long 
    lpHookProc_LpkDrawTextEx As Long 
    lpHookProc_LpkEditControl As Long 
End Type 

Public DLLhwnd As Long, DLLFunDre As Long 
Public LpkHooksInfo As LpkHooksSetting 

Public Function GetLocalProcAdress(ByVal lpProc As Long) As Long 
    GetLocalProcAdress = lpProc 
End Function 

Function HookProc1(ByVal a1 As Long, ByVal a2 As Long, ByVal a3 As Long, ByVal a4 As Long, ByVal a5 As Long, ByVal a6 As Long, ByVal a7 As Long, ByVal a8 As Long, ByVal a9 As Long, ByVal a10 As Long) As Long 
    HookProc1 = 0 
End Function 
```
运行发现窗体中标题栏和按钮上的文字都没有了，因为函数LpkDrawTextEx已经被替换成函数HookProc1了。函数LpkDrawTextEx有10个参数，其中几个是字符串指针，可以用来截获窗体要显示的文字，然后改成另一种语言的文字。
## 2.3. Inline Hook（API Hook）
使用汇编代码替代函数开始的汇编代码来获取控制权。
### 2.3.1. 步骤
* 定位函数
* 修改页面属性为可读可写
* 保存现场（函数初始或中间）
* 改写现场跳转到Hook函数

### 2.3.2. 注意事项
* 视情况需要进行脱钩以及再挂钩操作
* 对于不同的CPU，替换的汇编代码有所区别
* 替换汇编代码为非原子操作，多线程情况下可能出现异常导致崩溃，解决方法如下（参见MHook库）
    * 先上互斥锁
    * 先挂起其它线程并保证线程的IP指针不位于替换区域
## 2.4. IAT Hook
替换导入表中的函数地址来获取控制权。
## 2.5. SEH Hook
安装一个顶层SEH函数，并在函数开头插入触发异常代码，以此获取控制权。
## 2.6. 实现全局Hook
挂钩NtResumeThread函数，对于所有新创建的进程都进行挂钩操作。但是NtResumeThread并不是创建进程才调用，所以需要先枚举系统进程一次，将系统进程中NtResumeThread都挂钩上，这样，之后每次触发钩子时先判断NtResumeThread是否已经被挂钩，如果没有则是创建新进程的调用。
# 3. Ring0层Hook
## 3.1. IRP Hook
## 3.2. MSR Hook
## 3.3. SSDT Hook
编写驱动加载至内核空间，将驱动中的我们自己的函数地址替换到SSDT中，应用层调用API后，最后会拐到我们自己的函数中，达到Hook的目的。

系统有两张SSDT表，一个是导出的KeServiceDescriptorTable(ntoskrnl.exe)，一个是未导出的KeServiceDescriptorTableShadow(ntoskrnl.exe,win32k.sys)由于内核已经导出KeServiceDescriptorTable全局变量，所以编写程序的时候，不需要定位KeServiceDescriptorTable。KeServiceDescriptorTableShadow未导出，所以需要使用一些非公开的方法来定位此地址，通常都是采用硬性编码的，没有系统适应性。

SSDT的缺点在于在64位的系统下，基本无法工作，除非你能跨过微软的安全防护。目前来看，还没有人破解win8.1. 
# 4. Hook方式
## 4.1. 手动Hook
* Ring0 Hook
* Dll注入

## 4.2. Hook库
* EasyHook，支持Ring0（不够稳定）和Ring3，该库对多线程未进行处理
* Mhook，只支持Ring3

# 5. Hook注意事项
## 5.1. _declspec(naked)
就是告诉编译器，在编译的时候，不要优化代码，不要添加额外代码来控制堆栈平衡，一切代码都需要自己来写，防止破坏被Hook函数的堆栈或者导致堆栈不平衡。
```c
#define NAKED __declspec(naked)
void NAKED code(void)
{
    __asm{
        ret
    }
}
```
