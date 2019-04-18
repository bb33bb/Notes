<!-- TOC -->

- [1. 后缀](#1-后缀)
- [2. 前缀](#2-前缀)
- [3. Zw系列函数和Nt系列函数的区别](#3-zw系列函数和nt系列函数的区别)
    - [3.1. previous mode](#31-previous-mode)
    - [3.2. 用户模式（ntdll.dll中调用）](#32-用户模式ntdlldll中调用)
    - [3.3. 内核模式（ntoskrnl.exe中调用）](#33-内核模式ntoskrnlexe中调用)
    - [3.4. ntdll.dll和ntoskrnl.exe中ZW系列函数的区别](#34-ntdlldll和ntoskrnlexe中zw系列函数的区别)

<!-- /TOC -->
# 1. 后缀
* Ex（extend）：代表更新后的新型函数，且与原函数不兼容（可能会有两个Ex后缀）
* A（ascii）、W（wide）：A代表使用ASCII编码，W代表使用Unicode编码
* Callback：回调函数
# 2. 前缀
* w（word）：16位无符号值
* dw（dword）：32位无符号值
* H（handle）：句柄
* LP（long pointer）：32位指针
* P（pointer）：指针
* C（const）：固定值
* HWND：窗口句柄
# 3. Zw系列函数和Nt系列函数的区别
这里以NtOpenProcess和ZwOpenProcess为例，结合Windbg的lkd调试来说明。
## 3.1. previous mode
如果是从用户模式调用Native API则previous mode是用户态，如果是从内核模式调用Native API则previous mode是内核态。previous为用户态时Native API将对传递的参数进行严格的检查，而为内核态时则不会。
## 3.2. 用户模式（ntdll.dll中调用）
```x86asm
lkd> u ntdll!zwopenprocess l4
ntdll!ZwOpenProcess:
7c92d5fe b87a000000      mov     eax,7Ah           //函数调用号
7c92d603 ba0003fe7f      mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)     //函数地址
7c92d608 ff12            call    dword ptr [edx]
7c92d60a c21000          ret     10h

lkd> u ntdll!ntopenprocess l4
ntdll!ZwOpenProcess:
7c92d5fe b87a000000      mov     eax,7Ah
7c92d603 ba0003fe7f      mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)
7c92d608 ff12            call    dword ptr [edx]
7c92d60a c21000          ret     10h
```
二者没有任何区别，都是通过SSDT服务表，通过系统服务调度程序KiSystemService调用ntoskrnl.exe中的中断处理程序Nt系列函数。

在用户模式（ntdll.lib）下，二者没有任何区别，均为通过设置系统服务表中的索引和在堆栈中设置参数，经由SYSENTER指令进入内核态（而不是象w2k中通过int 0x2e中断），并最终由KiSystemService跳转到KiServiceTable对应的系统服务例程中。由于是从用户模式进入内核模式，因此代码会严格检查用户空间传入的参数。
## 3.3. 内核模式（ntoskrnl.exe中调用）
```x86asm
lkd> u nt!zwopenprocess l6
nt!ZwOpenProcess:
804e4c0a b87a000000      mov     eax,7Ah
804e4c0f 8d542404        lea     edx,[esp+4]
804e4c13 9c              pushfd
804e4c14 6a08            push    8
804e4c16 e8b69bffff      call    nt!KiSystemService (804de7d1)
804e4c1b c21000          ret     10h

lkd> u nt!ntopenprocess l20
nt!NtOpenProcess:
80582702 68c4000000      push    0C4h
80582707 68e8524f80      push    offset nt!ObWatchHandles+0x25c (804f52e8)
8058270c e87217f6ff      call    nt!_SEH_prolog (804e3e83)
80582711 33f6            xor     esi,esi
80582713 8975d4          mov     dword ptr [ebp-2Ch],esi
80582716 33c0            xor     eax,eax
80582718 8d7dd8          lea     edi,[ebp-28h]
8058271b ab              stos    dword ptr es:[edi]
8058271c 64a124010000    mov     eax,dword ptr fs:[00000124h]
80582722 8a8040010000    mov     al,byte ptr [eax+140h]
80582728 8845cc          mov     byte ptr [ebp-34h],al
8058272b 84c0            test    al,al
8058272d 0f840e7b0100    je      nt!NtOpenProcess+0xc0 (8059a241)
80582733 8975fc          mov     dword ptr [ebp-4],esi
80582736 a1d48e5680      mov     eax,dword ptr [nt!MmUserProbeAddress (80568ed4)]
8058273b 8b4d08          mov     ecx,dword ptr [ebp+8]
8058273e 3bc8            cmp     ecx,eax
80582740 0f8315170800    jae     nt!NtOpenProcess+0x40 (80603e5b)
80582746 8b01            mov     eax,dword ptr [ecx]
80582748 8901            mov     dword ptr [ecx],eax
8058274a 8b5d10          mov     ebx,dword ptr [ebp+10h]
8058274d f6c303          test    bl,3
80582750 0f850c170800    jne     nt!NtOpenProcess+0x4e (80603e62)
80582756 a1d48e5680      mov     eax,dword ptr [nt!MmUserProbeAddress (80568ed4)]
8058275b 3bd8            cmp     ebx,eax
8058275d 0f8309170800    jae     nt!NtOpenProcess+0x5c (80603e6c)
80582763 397308          cmp     dword ptr [ebx+8],esi
80582766 0f9545e6        setne   byte ptr [ebp-1Ah]
8058276a 8b4b0c          mov     ecx,dword ptr [ebx+0Ch]
8058276d 894dc8          mov     dword ptr [ebp-38h],ecx
80582770 8b4d14          mov     ecx,dword ptr [ebp+14h]
80582773 3bce            cmp     ecx,esi
```
Nt系列函数是直接调用对应功能代码，而Zw系列函数是在ring 0下通过SSDT服务表，KiSystemService调用ntoskrnl.exe中的中断处理程序Nt系列函数。

在内核模式（nooskrnl.lib）下，调用Nt系列函数时不会改变previous mode的状态，调用Zw系列函数时会将previous mode改为内核态。因此在进行Kernel Mode Driver开发时可以使用Zw系列函数来避免额外的参数列表检查，提高效率。
## 3.4. ntdll.dll和ntoskrnl.exe中ZW系列函数的区别
* ntdll.dll：传入服务号到eax——>调用SharedUserData!SystemCallStub (7ffe0300)函数->KiSystemService---->Nt系列函数(ntoskrnl)
* ntoskrnl.exe：传入服务号eax——>KiSystemService--->Nt系列函数(ntoskrnl)

