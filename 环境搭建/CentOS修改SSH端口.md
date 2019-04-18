<!-- TOC -->

- [1. 关闭selinux](#1-关闭selinux)
- [2. 修改端口](#2-修改端口)
- [3. 修改iptables](#3-修改iptables)

<!-- /TOC -->
# 1. 关闭selinux
修改`/etc/sysconfig/selinux`，将`SELINUX=enforcing`改为`SELINUX=disabled`，重启生效
# 2. 修改端口
修改`/etc/ssh/sshd_config`，将`# Port 22`修改为`Port xx`，重启SSH服务生效
# 3. 修改iptables
修改`/etc/sysconfig/iptables`，加一行`-A INPUT -m state --state NEW -m tcp -p tcp --dport xx -j ACCEPT`，重启iptables服务生效

