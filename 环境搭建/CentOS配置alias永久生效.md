<!-- TOC -->

- [1. alias（别名）](#1-alias别名)
- [2. 一些好用的alisa](#2-一些好用的alisa)

<!-- /TOC -->
# 1. alias（别名）
直接执行alias仅对当前shell环境有效，若需永久生效，如下所示
```bash
#单用户
#vi ~/.bashrc
#全局
vi /etc/bashrc
#alias ...
#将命令写入文件
#执行以下命令或者重启以生效
#单用户
#source ~/.bashrc
source /etc/bashrc
```
# 2. 一些好用的alisa
```bash
alias ll='ls -lht'     #按修改时间逆序列出文件 
alias la='ls -lhta'    #按修改时间逆序列出所有文件 
alias size='f(){ du -sh $1* | sort -hr; }; f'   #列出文件/目录大小
alias sek='f(){ find / -name $1; }; f'       #在根目录查找文件 
alias sekc='f(){ find ./ -name $1; }; f'     #在当前目录查找文件
alias portopen='f(){ /sbin/iptables -I INPUT -p tcp --dport $1 -j ACCEPT; }; f'   #打开端口
alias portclose='f(){ /sbin/iptables -I INPUT -p tcp --dport $1 -j DROP; }; f'    #关闭端口
alias www='f(){ python -m SimpleHTTPServer $1; }; f'     #开启http服务
alias auto='systemctl list-unit-files --type=service | grep enabled | more'    #列出启动项
alias now='date "+%Y-%m-%d %H:%M:%S"'    #当前时间
alias dkrnet='docker stats --no-stream | sort -k8 -hr | more'    #查看docker使用情况 
alias untar='tar xvf '     #解压tar
alias unjar='jar xvf '     #解压jar
alias ipe='curl ipinfo.io/ip'    #查看公网IP地址
```

