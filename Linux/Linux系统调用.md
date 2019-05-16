<!-- TOC -->

- [1. 系统调用](#1-系统调用)
- [2. Linux系统调用](#2-linux系统调用)
    - [2.1. 例子：sys_write](#21-例子sys_write)
- [3. Linux系统调用号列表](#3-linux系统调用号列表)

<!-- /TOC -->
# 1. 系统调用
在计算机中，系统调用（system call），又称为系统呼叫，指运行在使用者空间的程序向操作系统内核请求需要更高权限运行的服务。系统调用提供了用户程序与操作系统之间的接口。大多数系统交互式操作需求在内核态执行。如设备IO操作或者进程间通信。
# 2. Linux系统调用
Linux的系统调用通过int 80h实现，用系统调用号来区分入口函数。操作系统实现系统调用的基本过程是：
* 应用程序调用库函数（API）
* API将系统调用号存入EAX，参数存入其它寄存器，然后通过中断调用使系统进入内核态
* 内核中的中断处理函数根据系统调用号，调用对应的内核函数（系统调用）
* 系统调用完成相应功能，将返回值存入EAX，返回到中断处理函数
* 中断处理函数返回到API中
* API将EAX返回给应用程序
## 2.1. 例子：sys_write
sys_write(unsigned int fd, const char * buf, size_t count)
```x86asm
[section .data]
    strHello db "Hello, world!",0Ah
    STRLEN equ $ - strHello
[section .text]
    global _start
    _start:
        mov edx,STRLEN    ;对应参数count
        mov ecx,strHello  ;对应参数buf
        mov ebx,1         ;对应参数fd，fd = 1，在linux中对应于stdout，指的是显示屏
        mov eax,4         ;系统调用号为4，sys_write
        int 0x80
        mov ebx,0         ;参数为0，exit(0)
        mov eax,1         ;系统调用号为1，sys_exit
        int 0x80 
```
# 3. Linux系统调用号列表
|%eax|%eax-hex|Name|Source|%ebx|%ecx|%edx|%esx|%edi|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|0X1|sys_exit|kernel/exit.c|int|-|-|-|-|
|2|0X2|sys_fork|arch/i386/kernel/process.c|struct pt_regs|-|-|-|-|
|3|0X3|sys_read|fs/read_write.c|unsigned int|char *|size_t|-|-|
|4|0X4|sys_write|fs/read_write.c|unsigned int|const char *|size_t|-|-|
|5|0X5|sys_open|fs/open.c|const char *|int|int|-|-|
|6|0X6|sys_close|fs/open.c|unsigned int|-|-|-|-|
|7|0X7|sys_waitpid|kernel/exit.c|pid_t|unsigned int *|int|-|-|
|8|0X8|sys_creat|fs/open.c|const char *|int|-|-|-|
|9|0X9|sys_link|fs/namei.c|const char *|const char *|-|-|-|
|10|0XA|sys_unlink|fs/namei.c|const char *|-|-|-|-|
|11|0XB|sys_execve|arch/i386/kernel/process.c|struct pt_regs|-|-|-|-|
|12|0XC|sys_chdir|fs/open.c|const char *|-|-|-|-|
|13|0XD|sys_time|kernel/time.c|int *|-|-|-|-|
|14|0XE|sys_mknod|fs/namei.c|const char *|int|dev_t|-|-|
|15|0XF|sys_chmod|fs/open.c|const char *|mode_t|-|-|-|
|16|0X10|sys_lchown|fs/open.c|const char *|uid_t|gid_t|-|-|
|18|0X12|sys_stat|fs/stat.c|char *|struct __old_kernel_stat *|-|-|-|
|19|0X13|sys_lseek|fs/read_write.c|unsigned int|off_t|unsigned int|-|-|
|20|0X14|sys_getpid|kernel/sched.c|-|-|-|-|-|
|21|0X15|sys_mount|fs/super.c|char *|char *|char *|-|-|
|22|0X16|sys_oldumount|fs/super.c|char *|-|-|-|-|
|23|0X17|sys_setuid|kernel/sys.c|uid_t|-|-|-|-|
|24|0X18|sys_getuid|kernel/sched.c|-|-|-|-|-|
|25|0X19|sys_stime|kernel/time.c|int *|-|-|-|-|
|26|0X1A|sys_ptrace|arch/i386/kernel/ptrace.c|long|long|long|long|-|
|27|0X1B|sys_alarm|kernel/sched.c|unsigned int|-|-|-|-|
|28|0X1C|sys_fstat|fs/stat.c|unsigned int|struct __old_kernel_stat *|-|-|-|
|29|0X1D|sys_pause|arch/i386/kernel/sys_i386.c|-|-|-|-|-|
|30|0X1E|sys_utime|fs/open.c|char *|struct utimbuf *|-|-|-|
|33|0X21|sys_access|fs/open.c|const char *|int|-|-|-|
|34|0X22|sys_nice|kernel/sched.c|int|-|-|-|-|
|36|0X24|sys_sync|fs/buffer.c|-|-|-|-|-|
|37|0X25|sys_kill|kernel/signal.c|int|int|-|-|-|
|38|0X26|sys_rename|fs/namei.c|const char *|const char *|-|-|-|
|39|0X27|sys_mkdir|fs/namei.c|const char *|int|-|-|-|
|40|0X28|sys_rmdir|fs/namei.c|const char *|-|-|-|-|
|41|0X29|sys_dup|fs/fcntl.c|unsigned int|-|-|-|-|
|42|0X2A|sys_pipe|arch/i386/kernel/sys_i386.c|unsigned long *|-|-|-|-|
|43|0X2B|sys_times|kernel/sys.c|struct tms *|-|-|-|-|
|45|0X2D|sys_brk|mm/mmap.c|unsigned long|-|-|-|-|
|46|0X2E|sys_setgid|kernel/sys.c|gid_t|-|-|-|-|
|47|0X2F|sys_getgid|kernel/sched.c|-|-|-|-|-|
|48|0X30|sys_signal|kernel/signal.c|int|__sighandler_t|-|-|-|
|49|0X31|sys_geteuid|kernel/sched.c|-|-|-|-|-|
|50|0X32|sys_getegid|kernel/sched.c|-|-|-|-|-|
|51|0X33|sys_acct|kernel/acct.c|const char *|-|-|-|-|
|52|0X34|sys_umount|fs/super.c|char *|int|-|-|-|
|54|0X36|sys_ioctl|fs/ioctl.c|unsigned int|unsigned int|unsigned long|-|-|
|55|0X37|sys_fcntl|fs/fcntl.c|unsigned int|unsigned int|unsigned long|-|-|
|57|0X39|sys_setpgid|kernel/sys.c|pid_t|pid_t|-|-|-|
|59|0X3B|sys_olduname|arch/i386/kernel/sys_i386.c|struct oldold_utsname *|-|-|-|-|
|60|0X3C|sys_umask|kernel/sys.c|int|-|-|-|-|
|61|0X3D|sys_chroot|fs/open.c|const char *|-|-|-|-|
|62|0X3E|sys_ustat|fs/super.c|dev_t|struct ustat *|-|-|-|
|63|0X3F|sys_dup2|fs/fcntl.c|unsigned int|unsigned int|-|-|-|
|64|0X40|sys_getppid|kernel/sched.c|-|-|-|-|-|
|65|0X41|sys_getpgrp|kernel/sys.c|-|-|-|-|-|
|66|0X42|sys_setsid|kernel/sys.c|-|-|-|-|-|
|67|0X43|sys_sigaction|arch/i386/kernel/signal.c|int|const struct old_sigaction *|struct old_sigaction *|-|-|
|68|0X44|sys_sgetmask|kernel/signal.c|-|-|-|-|-|
|69|0X45|sys_ssetmask|kernel/signal.c|int|-|-|-|-|
|70|0X46|sys_setreuid|kernel/sys.c|uid_t|uid_t|-|-|-|
|71|0X47|sys_setregid|kernel/sys.c|gid_t|gid_t|-|-|-|
|72|0X48|sys_sigsuspend|arch/i386/kernel/signal.c|int|int|old_sigset_t|-|-|
|73|0X49|sys_sigpending|kernel/signal.c|old_sigset_t *|-|-|-|-|
|74|0X4A|sys_sethostname|kernel/sys.c|char *|int|-|-|-|
|75|0X4B|sys_setrlimit|kernel/sys.c|unsigned int|struct rlimit *|-|-|-|
|76|0X4C|sys_getrlimit|kernel/sys.c|unsigned int|struct rlimit *|-|-|-|
|77|0X4D|sys_getrusage|kernel/sys.c|int|struct rusage *|-|-|-|
|78|0X4E|sys_gettimeofday|kernel/time.c|struct timeval *|struct timezone *|-|-|-|
|79|0X4F|sys_settimeofday|kernel/time.c|struct timeval *|struct timezone *|-|-|-|
|80|0X50|sys_getgroups|kernel/sys.c|int|gid_t *|-|-|-|
|81|0X51|sys_setgroups|kernel/sys.c|int|gid_t *|-|-|-|
|82|0X52|old_select|arch/i386/kernel/sys_i386.c|struct sel_arg_struct *|-|-|-|-|
|83|0X53|sys_symlink|fs/namei.c|const char *|const char *|-|-|-|
|84|0X54|sys_lstat|fs/stat.c|char *|struct __old_kernel_stat *|-|-|-|
|85|0X55|sys_readlink|fs/stat.c|const char *|char *|int|-|-|
|86|0X56|sys_uselib|fs/exec.c|const char *|-|-|-|-|
|87|0X57|sys_swapon|mm/swapfile.c|const char *|int|-|-|-|
|88|0X58|sys_reboot|kernel/sys.c|int|int|int|void *|-|
|89|0X59|old_readdir|fs/readdir.c|unsigned int|void *|unsigned int|-|-|
|90|0X5A|old_mmap|arch/i386/kernel/sys_i386.c|struct mmap_arg_struct *|-|-|-|-|
|91|0X5B|sys_munmap|mm/mmap.c|unsigned long|size_t|-|-|-|
|92|0X5C|sys_truncate|fs/open.c|const char *|unsigned long|-|-|-|
|93|0X5D|sys_ftruncate|fs/open.c|unsigned int|unsigned long|-|-|-|
|94|0X5E|sys_fchmod|fs/open.c|unsigned int|mode_t|-|-|-|
|95|0X5F|sys_fchown|fs/open.c|unsigned int|uid_t|gid_t|-|-|
|96|0X60|sys_getpriority|kernel/sys.c|int|int|-|-|-|
|97|0X61|sys_setpriority|kernel/sys.c|int|int|int|-|-|
|99|0X63|sys_statfs|fs/open.c|const char *|struct statfs *|-|-|-|
|100|0X64|sys_fstatfs|fs/open.c|unsigned int|struct statfs *|-|-|-|
|101|0X65|sys_ioperm|arch/i386/kernel/ioport.c|unsigned long|unsigned long|int|-|-|
|102|0X66|sys_socketcall|net/socket.c|int|unsigned long *|-|-|-|
|103|0X67|sys_syslog|kernel/printk.c|int|char *|int|-|-|
|104|0X68|sys_setitimer|kernel/itimer.c|int|struct itimerval *|struct itimerval *|-|-|
|105|0X69|sys_getitimer|kernel/itimer.c|int|struct itimerval *|-|-|-|
|106|0X6A|sys_newstat|fs/stat.c|char *|struct stat *|-|-|-|
|107|0X6B|sys_newlstat|fs/stat.c|char *|struct stat *|-|-|-|
|108|0X6C|sys_newfstat|fs/stat.c|unsigned int|struct stat *|-|-|-|
|109|0X6D|sys_uname|arch/i386/kernel/sys_i386.c|struct old_utsname *|-|-|-|-|
|110|0X6E|sys_iopl|arch/i386/kernel/ioport.c|unsigned long|-|-|-|-|
|111|0X6F|sys_vhangup|fs/open.c|-|-|-|-|-|
|112|0X70|sys_idle|arch/i386/kernel/process.c|-|-|-|-|-|
|113|0X71|sys_vm86old|arch/i386/kernel/vm86.c|unsigned long|struct vm86plus_struct *|-|-|-|
|114|0X72|sys_wait4|kernel/exit.c|pid_t|unsigned long *|int options|struct rusage *|-|
|115|0X73|sys_swapoff|mm/swapfile.c|const char *|-|-|-|-|
|116|0X74|sys_sysinfo|kernel/info.c|struct sysinfo *|-|-|-|-|
|117|0X75|sys_ipc (*Note)|arch/i386/kernel/sys_i386.c|uint|int|int|int|void *|
|118|0X76|sys_fsync|fs/buffer.c|unsigned int|-|-|-|-|
|119|0X77|sys_sigreturn|arch/i386/kernel/signal.c|unsigned long|-|-|-|-|
|120|0X78|sys_clone|arch/i386/kernel/process.c|struct pt_regs|-|-|-|-|
|121|0X79|sys_setdomainname|kernel/sys.c|char *|int|-|-|-|
|122|0X7A|sys_newuname|kernel/sys.c|struct new_utsname *|-|-|-|-|
|123|0X7B|sys_modify_ldt|arch/i386/kernel/ldt.c|int|void *|unsigned long|-|-|
|124|0X7C|sys_adjtimex|kernel/time.c|struct timex *|-|-|-|-|
|125|0X7D|sys_mprotect|mm/mprotect.c|unsigned long|size_t|unsigned long|-|-|
|126|0X7E|sys_sigprocmask|kernel/signal.c|int|old_sigset_t *|old_sigset_t *|-|-|
|127|0X7F|sys_create_module|kernel/module.c|const char *|size_t|-|-|-|
|128|0X80|sys_init_module|kernel/module.c|const char *|struct module *|-|-|-|
|129|0X81|sys_delete_module|kernel/module.c|const char *|-|-|-|-|
|130|0X82|sys_get_kernel_syms|kernel/module.c|struct kernel_sym *|-|-|-|-|
|131|0X83|sys_quotactl|fs/dquot.c|int|const char *|int|caddr_t|-|
|132|0X84|sys_getpgid|kernel/sys.c|pid_t|-|-|-|-|
|133|0X85|sys_fchdir|fs/open.c|unsigned int|-|-|-|-|
|134|0X86|sys_bdflush|fs/buffer.c|int|long|-|-|-|
|135|0X87|sys_sysfs|fs/super.c|int|unsigned long|unsigned long|-|-|
|136|0X88|sys_personality|kernel/exec_domain.c|unsigned long|-|-|-|-|
|138|0X8A|sys_setfsuid|kernel/sys.c|uid_t|-|-|-|-|
|139|0X8B|sys_setfsgid|kernel/sys.c|gid_t|-|-|-|-|
|140|0X8C|sys_llseek|fs/read_write.c|unsigned int|unsigned long|unsigned long|loff_t *|unsigned int|
|141|0X8D|sys_getdents|fs/readdir.c|unsigned int|void *|unsigned int|-|-|
|142|0X8E|sys_select|fs/select.c|int|fd_set *|fd_set *|fd_set *|struct timeval *|
|143|0X8F|sys_flock|fs/locks.c|unsigned int|unsigned int|-|-|-|
|144|0X90|sys_msync|mm/filemap.c|unsigned long|size_t|int|-|-|
|145|0X91|sys_readv|fs/read_write.c|unsigned long|const struct iovec *|unsigned long|-|-|
|146|0X92|sys_writev|fs/read_write.c|unsigned long|const struct iovec *|unsigned long|-|-|
|147|0X93|sys_getsid|kernel/sys.c|pid_t|-|-|-|-|
|148|0X94|sys_fdatasync|fs/buffer.c|unsigned int|-|-|-|-|
|149|0X95|sys_sysctl|kernel/sysctl.c|struct __sysctl_args *|-|-|-|-|
|150|0X96|sys_mlock|mm/mlock.c|unsigned long|size_t|-|-|-|
|151|0X97|sys_munlock|mm/mlock.c|unsigned long|size_t|-|-|-|
|152|0X98|sys_mlockall|mm/mlock.c|int|-|-|-|-|
|153|0X99|sys_munlockall|mm/mlock.c|-|-|-|-|-|
|154|0X9A|sys_sched_setparam|kernel/sched.c|pid_t|struct sched_param *|-|-|-|
|155|0X9B|sys_sched_getparam|kernel/sched.c|pid_t|struct sched_param *|-|-|-|
|156|0X9C|sys_sched_setscheduler|kernel/sched.c|pid_t|int|struct sched_param *|-|-|
|157|0X9D|sys_sched_getscheduler|kernel/sched.c|pid_t|-|-|-|-|
|158|0X9E|sys_sched_yield|kernel/sched.c|-|-|-|-|-|
|159|0X9F|sys_sched_get_priority_max|kernel/sched.c|int|-|-|-|-|
|160|0XA0|sys_sched_get_priority_min|kernel/sched.c|int|-|-|-|-|
|161|0XA1|sys_sched_rr_get_interval|kernel/sched.c|pid_t|struct timespec *|-|-|-|
|162|0XA2|sys_nanosleep|kernel/sched.c|struct timespec *|struct timespec *|-|-|-|
|163|0XA3|sys_mremap|mm/mremap.c|unsigned long|unsigned long|unsigned long|unsigned long|-|
|164|0XA4|sys_setresuid|kernel/sys.c|uid_t|uid_t|uid_t|-|-|
|165|0XA5|sys_getresuid|kernel/sys.c|uid_t *|uid_t *|uid_t *|-|-|
|166|0XA6|sys_vm86|arch/i386/kernel/vm86.c|struct vm86_struct *|-|-|-|-|
|167|0XA7|sys_query_module|kernel/module.c|const char *|int|char *|size_t|size_t *|
|168|0XA8|sys_poll|fs/select.c|struct pollfd *|unsigned int|long|-|-|
|169|0XA9|sys_nfsservctl|fs/filesystems.c|int|void *|void *|-|-|
|170|0XAA|sys_setresgid|kernel/sys.c|gid_t|gid_t|gid_t|-|-|
|171|0XAB|sys_getresgid|kernel/sys.c|gid_t *|gid_t *|gid_t *|-|-|
|172|0XAC|sys_prctl|kernel/sys.c|int|unsigned long|unsigned long|unsigned long|unsigned long|
|173|0XAD|sys_rt_sigreturn|arch/i386/kernel/signal.c|unsigned long|-|-|-|-|
|174|0XAE|sys_rt_sigaction|kernel/signal.c|int|const struct sigaction *|struct sigaction *|size_t|-|
|175|0XAF|sys_rt_sigprocmask|kernel/signal.c|int|sigset_t *|sigset_t *|size_t|-|
|176|0XB0|sys_rt_sigpending|kernel/signal.c|sigset_t *|size_t|-|-|-|
|177|0XB1|sys_rt_sigtimedwait|kernel/signal.c|const sigset_t *|siginfo_t *|const struct timespec *|size_t|-|
|178|0XB2|sys_rt_sigqueueinfo|kernel/signal.c|int|int|siginfo_t *|-|-|
|179|0XB3|sys_rt_sigsuspend|arch/i386/kernel/signal.c|sigset_t *|size_t|-|-|-|
|180|0XB4|sys_pread|fs/read_write.c|unsigned int|char *|size_t|loff_t|-|
|181|0XB5|sys_pwrite|fs/read_write.c|unsigned int|const char *|size_t|loff_t|-|
|182|0XB6|sys_chown|fs/open.c|const char *|uid_t|gid_t|-|-|
|183|0XB7|sys_getcwd|fs/dcache.c|char *|unsigned long|-|-|-|
|184|0XB8|sys_capget|kernel/capability.c|cap_user_header_t|cap_user_data_t|-|-|-|
|185|0XB9|sys_capset|kernel/capability.c|cap_user_header_t|const cap_user_data_t|-|-|-|
|186|0XBA|sys_sigaltstack|arch/i386/kernel/signal.c|const stack_t *|stack_t *|-|-|-|
|187|0XBB|sys_sendfile|mm/filemap.c|int|int|off_t *|size_t|-|
|190|0XBE|sys_vfork|arch/i386/kernel/process.c|struct pt_regs|-|-|-|-|