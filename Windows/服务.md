<!-- TOC -->

- [1. 服务是什么](#1-服务是什么)
- [2. 服务的特性](#2-服务的特性)
- [3. 服务的运行方式](#3-服务的运行方式)
- [4. 编程实现](#4-编程实现)
    - [4.1. 主体部分](#41-主体部分)
    - [4.2. 安装服务程序](#42-安装服务程序)

<!-- /TOC -->
# 1. 服务是什么    
提供了像Unix下后台程序Daemons（守护进程）的等价物，而且使得创建能够代表权限低的用户进行权限高的操作的程序成为可能。 服务大都是由服务控制程序在注册表中维护的一个信息数据库来管理的，每个服务在HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services中都可以找到相应的一个关键项。
# 2. 服务的特性
* 服务可以被指定为自启动
* 服务可以在任何用户登录前开始运行，我们可以在服务启动时加入杀防火墙的代码
* 服务是运行在后台的
# 3. 服务的运行方式
* WIN32_SHARE_PROCESS：这种类型将这个服务的代码保存到一个DLL中，并在一个进程中组合多个不同的服务，进程中表现为svchost.exe，Windows服务通常以这种方式运行
* WIN32_OWN_PROCESS：在一个exe文件中保存代码，并且作为一个独立进程运行
* KERNEL_DRIVER：加载代码到内核中执行
# 4. 编程实现
## 4.1. 主体部分
```c
void main(){
    SERVICE_TABLE_ENTRY ServiceTable[] = {
        {"scuhkr", BDServiceMain},  /*名称、服务函数，这里我们只安装了一个服务*/
        {NULL, NULL}                /*最后一个成员必须为NULL，称为哨兵*/
    };
    //连接到服务控制管理器
    StartServiceCtrlDispatcher(ServiceTable);
}
//lpszArgv参数个数，该数组第一个的参数指定了服务名，可以在后面被StartService()来调用，参数必须如此格式
void WINAPI BDServiceMain(DWORD dwArgc, LPTSTR *lpszArgv)
{
    DWORD dwThreadId;  //存放线程ID
    //通过RegisterServiceCtrlHandler()与服务控制程序建立一个通信的协议
    //BDHandler()是我们的服务控制程序，它被可以被用来开始，暂停，恢复，停止服务等控制操作
    if (!(ServiceStatusHandle = RegisterServiceCtrlHandler("scuhkr", BDHandler)))
        return;
    ServiceStatus.dwServiceType  = SERVICE_WIN32_OWN_PROCESS;    //表示该服务私有
    ServiceStatus.dwCurrentState  = SERVICE_START_PENDING;     //初始化服务，正在开始
    //服务可以接受的请求，这里我们只接受停止服务请求和暂停恢复请求
    ServiceStatus.dwControlsAccepted  = SERVICE_ACCEPT_STOP | SERVICE_ACCEPT_PAUSE_CONTINUE;
    //下面几个一般我们不大关心，全为0
    ServiceStatus.dwServiceSpecificExitCode = 0;
    ServiceStatus.dwWin32ExitCode = 0;
    ServiceStatus.dwCheckPoint = 0;
    ServiceStatus.dwWaitHint = 0;    //必须调用SetServiceStatus()来响应服务控制程序的每次请求通知
    SetServiceStatus(ServiceStatusHandle, &ServiceStatus);    //开始运行服务
    ServiceStatus.dwCurrentState = SERVICE_RUNNING;
    ServiceStatus.dwCheckPoint = 0;
    ServiceStatus.dwWaitHint = 0;
    SetServiceStatus(ServiceStatusHandle, &ServiceStatus);    //我们用一个事件对象来控制服务的同步
    if (!(hEvent=CreateEvent(NULL, FALSE, FALSE, NULL)))
        return;
    ServiceStatus.dwCurrentState = SERVICE_START_PENDING;
    ServiceStatus.dwCheckPoint = 0;
    ServiceStatus.dwWaitHint = 0;
    SetServiceStatus(ServiceStatusHandle, &ServiceStatus);    //开线程来启动我们的后门程序
    if (!(hThread=CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)MainFn, (LPVOID)0, 0, &dwThreadId)))
        ServiceStatus.dwCurrentState = SERVICE_RUNNING;
    ServiceStatus.dwCheckPoint = 0; 
    ServiceStatus.dwWaitHint = 0;
    WaitForSingleObject(hEvent, INFINITE);
    CloseHandle(hThread);
    ExitThread(dwThreadId);
    CloseHandle(hEvent);
    return;
}
//服务控制函数
void WINAPI BDHandler(DWORD dwControl){
    switch(dwControl){
        case SERVICE_CONTROL_STOP:            //等待后门程序的停止
            ServiceStatus.dwCurrentState = SERVICE_STOP_PENDING;
            ServiceStatus.dwCheckPoint   = 0;
            ServiceStatus.dwWaitHint     = 0;
            SetServiceStatus(ServiceStatusHandle, &ServiceStatus);            //设时间为激发状态，等待下一个事件的到来
            SetEvent(hEvent);
            ServiceStatus.dwCurrentState = SERVICE_STOP;
            ServiceStatus.dwCheckPoint   = 0;
            ServiceStatus.dwWaitHint     = 0;            //停止
            SetServiceStatus(ServiceStatusHandle, &ServiceStatus);
            break;
        default:
            break;
    }
}
//后门程序
DWORD WINAPI MainFn(LPVOID lpParam){    }
```
## 4.2. 安装服务程序
```c
//InstallService.cpp
void main(){
    SC_HANDLE hSCManager = NULL,  //服务控制管理器句柄
    hService = NULL;     //服务句柄
    char szSysPath[MAX_PATH],
    szExePath[MAX_PATH];   //我们要把我们后台执行的程序放在这里，一般就是在\\admin$\\system32\里，隐蔽性高
    //NULL表明是本地主机；要打开的服务控制管理数据库，默认为空；创建权限
    if ((hSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_CREATE_SERVICE))==NULL){
        pirntf("OpenSCManager failed\n");
        return;
    }
    GetSystemDirectory(szSysPath, MAX_PATH); //获得系统目录，也就是system32里面，隐蔽起来
    strcpy(szExePath, szSysPath);
    strcat(szExePath, "scuhkr.exe");  //应用程序绝对路径
    //指向服务控制管理数据库的句柄，服务名，显示用的服务名，所有访问权限，私有类型，自启动类型，忽略错误处理，应用程序路径
    if ((hService=CreateService(hSCManager, "scuhkr", "scuhkr backdoor service", SERVICE_ALL_ACCESS, SERVICE_WIN32_OWN_PROCESS,
                                SERVICE_DEMAND_START, SERVICE_ERROR_IGNORE, szExePath, NULL, NULL, NULL, NULL, NULL)) == NULL){
        printf("%d\n", GetLastError());
        return;
    }
    //让服务马上运行
    if(StartService(hService, 0, NULL) == FALSE){
        printf("StartService failed: %d\n", GetLastError());
        return;
    }
    printf(“Install service successfully\n ”);
    CloseServiceHandle(hService);  //关闭服务句柄
    CloseServiceHandle(hSCManager); //关闭服务管理数据库句柄
}
```

