<!-- TOC -->

- [1. Python3](#1-python3)
- [2. 简易安装第三方SS、SSR](#2-简易安装第三方ssssr)
- [3. MySQL](#3-mysql)
- [4. metasploit](#4-metasploit)

<!-- /TOC -->
# 1. Python3
```bash
yum -y groupinstall 'Development Tools'
yum -y install zlib-devel bzip2-devel  openssl-devel ncurses-devel
wget --no-check-certificate https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz
sudo mkdir /usr/local/python3
tar -zxvf Python-3.6.5.tgz
cd Python-3.6.5/
./configure --prefix=/usr/local/python3 --enable-optimizations
make
make install
ln -s /usr/local/python3/bin/python3.6 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
pip3 install --upgrade pip
pip3 install pymysql
```
# 2. 简易安装第三方SS、SSR
```bash
#安装SS
yum -y install git
#apt-get update && apt-get install -y git
git clone https://github.com/flyzy2005/ss-fly
ss-fly/ss-fly.sh -i flyzy2005.com 1024
ss-fly/ss-fly.sh -bbr

#修改配置文件：vi /etc/shadowsocks.json
#停止ss服务：ssserver -c /etc/shadowsocks.json -d stop
#启动ss服务：ssserver -c /etc/shadowsocks.json -d start
#重启ss服务：ssserver -c /etc/shadowsocks.json -d restart
#卸载SS：ss-fly/ss-fly.sh -uninstall

#安装SSR
yum -y install git
#apt-get update && apt-get install -y git
git clone https://github.com/flyzy2005/ss-fly
ss-fly/ss-fly.sh -ssr
ss-fly/ss-fly.sh -bbr

#启动：/etc/init.d/shadowsocks start
#停止：/etc/init.d/shadowsocks stop
#重启：/etc/init.d/shadowsocks restart
#状态：/etc/init.d/shadowsocks status
#配置文件路径：/etc/shadowsocks.json
#日志文件路径：/var/log/shadowsocks.log
#代码安装目录：/usr/local/shadowsocks
#卸载SSR：./shadowsocksR.sh uninstall

#判断BBR加速是否成功（返回值为net.ipv4.tcp_available_congestion_control = bbr cubic reno代表成功）
sysctl net.ipv4.tcp_available_congestion_control
```
# 3. MySQL
```bash
sudo yum localinstall https://repo.mysql.com//mysql80-community-release-el7-1.noarch.rpm
sudo yum install mysql-community-server
sudo service mysqld start
#查看初始随机密码 sudo grep 'temporary password' /var/log/mysqld.log
#登录 mysql -uroot -p
#如果无初始随机密码则在输入密码环节直接回车然后输入 flush privileges
#修改密码（use mysql）ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root';
#修改默认加密方式，修改文件/etc/my.cnf，加一行default_authentication_plugin=mysql_native_password;
#如果一开始不输入密码或输错密码都能连接MySQL server，修改文件 /etc/my.cnf，注释掉skip-grant-tables，重启MySQL服务
#后期如果忘记密码，可以通过skip-grant-tables配置跳过输入密码登录，之后修改密码（如果‘root’@'localhost'变为'root'@'%'，那么alter语句中的也要修改）
#配置MySQL允许外部访问：首先配置防火墙，之后use mysql;update user set host = '%' where user ='root';允许任何外部主机访问
#show global variables like 'port';可查看MySQL服务端口，如果看到的value为0，则说明没有使用密码登录，需要去修改my.cnf，port=3306，重启MySQL服务即可
```
# 4. metasploit
```bash
# 下载安装
yum install curl -y
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall
chmod 755 msfinstall
./msfinstall
# 初始化启动，此时会初始化数据库，由于metasploit使用的数据库无法与root用户绑定，第一次启动必须切换到非root用户，之后可以用root用户启动
adduser msf
su msf
cd /opt/metasploit-framework/bin
./msfconsole
# 如果手滑第一次启动用root用户启动了，执行以下命令重置
# msfdb reinit
```