---
title: Linux服务器入侵排查
date: 2018-08-16 15:39:03
categories: 安全排查
tags: Linux安全
---
## 一、针对情况
- 服务器上存在可疑进程，系统资源占用高（挖矿程序等）；
- 发现服务器对外大量发包（DDOS肉鸡）；
- 不正常的端口连接（反向shell等）；
- 服务器日志被恶意删除；
- ...

## 二、注意事项
- 对文件进行操作先备份文件再操作
- 不要vim等编辑器打开文件，会修改文件时间戳
- 拷贝源文件到服务器外需要附上md5校验
- 根据排查流程简单生成报告，svn提交

## 三、排查思路
1. 文件分析  
    a) 文件日期、新增文件、可疑/异常文件、最近使用文件、浏览器下载文件
    ​b) Webshell 排查与分析
    ​c) 核心应用关联目录文件分析

2. 进程分析  
    ​a) 当前活动进程 & 远程连接
    ​b) 启动进程&计划任务
    ​c) 进程工具分析
    ​ i. Windows:Pchunter
    ​ ii. Linux: Chkrootkit&Rkhunter

3. 系统信息  
    ​a) 环境变量
    ​b) 帐号信息
    ​c) History
    ​d) 系统配置文件
    
4. 日志分析  
    ​a) 操作系统日志
    ​ i. Windows: 事件查看器（eventvwr）
    ​ ii. Linux: /var/log/
    ​b) 应用日志分析
    ​ i. Access.log
    ​ ii. Error.log

## 四、分析排查

### 文件分析

1.敏感目录的文件分析
  
1）临时文件目录
如：/tmp/、/var/tmp 
木马或者异常文件可能隐藏在临时目录
2）命令目录
如：/usr/bin/、/usr/sbin/等，根据echo $PATH中的命令目录进行确定排查目录
常见手段为替换或修改这些目录的命令，隐藏恶意代码等。

针对可疑文件可以使用stat进行创建修改时间、访问时间的详细查看，若修改时间距离事件日期接近，有线性关联，说明可能被篡改或者其他。

​例如:
```
#​查看tmp目录下的文件
ls –alt /tmp/
#​查看文件修改、创建、访问时间
stat file
```

2.特殊权限的文件
特殊权限主要为777权限以及s位权限的文件，存在被篡改或替换的风险，也是后门木马程序的特征之一，需要留意。
分析方法，通过stat对可疑文件进行检查，修改时间是否为入侵阶段；file判断文件类型。
  
```
#查找777的权限的文件 
find / -type f -perm 4777
#查找s位权限的文件 
find / -type f -user root -group root \( -perm -04000 -o -perm -02000 \) -exec ls {} \;
```

### 进程分析

1.网络进程 
使用netstat 网络连接命令，分析可疑端口、可疑IP、可疑PID及程序进程
```
netstat -antlp | grep ESTABLISHED   查看已经建立的网络连接
netstat -antlp | grep LISTEN    检查可以监听端口
ip link | grep PROMISC　　正常网卡不应该存在promisc，如果存在可能有sniffer
lsof -i 列出所有的网络连接
arp -a　　查看arp记录是否正常
ifconfig -a　　查看网卡设置
```

2.获取进程
```
ps -aux　　查看进程
top        查看进程
lsof -p pid　　查看进程所打开的端口及文件
lsof -c 进程名　　查看关联文件
file  查看文件类型
strings 查看二进制文件内容可读字符
ps -aux | grep -v "\["   查看非系统进程

```

3.隐藏进程
简单对比ps -ef 与/proc的进程，后续使用rootkit进行检查较为可靠。

```
ps -ef | awk '{print $2}' | sort -n | uniq >1
ls /proc | sort -n |uniq >2
diff 1 2
```

4.后门排查
除以上文件、进程、系统 分析外，推荐工具：

​ chkrootkit rkhunter

rkhunter
```
apt-get install -y rkhunter
rkhunter --checkall 
#检查过程中需要回车确认，最后日志生成在/var/log/rkhunter，需重点关注日志中的warning等项目。
```


5.web服务检查
若无web服务，跳过此步。
一般如果网络边界做好控制，通常对外开放的仅是Web服务，那么需要先找到Webshell，可以通过如下途径：

1）检查web根目录
i.查看nginx或apache2的配置文件，获取root目录
ii.检查该目录下文件，以及浏览器访问情况
iii.分析可疑文件（关注jsp、php、asp、aspx等文件），是否存在webshell等后门
2)webshell查杀工具
使用河马shell
```
cd /opt/
wget -O hm-linux.tgz http://down.shellpub.com/hm/latest/hm-linux-amd64.tgz?version=1.4.2
tar xvf hm-linux.tgz
./hm scan 你的web目录
扫描完成之后结果会保存为result.csv文件，使用记事本或者excel打开查看
``` 
 
2）与测试环境目录做对比
```  
diff -r {生产dir} {测试dir} 
```
3）通过Access Log获取一些信息。

```
#伴随一些其他攻击特征
egrep '(select|script|acunetix|sqlmap)' /var/log/httpd/access_log
#页面访问排名前十的IP
cat access.log | cut -f1 -d " " | sort | uniq -c | sort -k 1 -r | head -10
#页面访问排名前十的URL
cat access.log | cut -f4 -d " " | sort | uniq -c | sort -k 1 -r | head -10
#查看最耗时的页面
cat access.log | sort -k 2 -n -r | head 10
#访问频次
awk '{print $1}' /var/log/nginx/access.log |sort -n |uniq -c |sort 
awk '{print $1}' /var/log/apache2/access.log |sort -n |uniq -c |sort 
```

### 系统信息

1.查看分析history，注意history是否篡改或删除  
a) wget 远程某主机（域名&IP）的远控文件； 
b) 尝试连接内网某主机（ssh scp），便于分析攻击者意图;
c) 打包某敏感数据或代码，tar zip 类命令
d) 对系统进行配置，包括命令修改、远控木马类，可找到攻击者关联信息…


2.查看分析用户相关分析  
​a) useradd userdel 的命令时间变化（stat），以及是否包含可疑信息
b) cat /etc/passwd 分析可疑帐号，可登录帐号
```
awk -F: '{if($3==0)print $1}' /etc/passwd  查看UID为0的帐号
cat /etc/passwd | grep -E "/bin/bash$" 查看能够登录的帐号
ls -l /etc/passwd　　查看passwd最后修改时间
awk -F: 'length($2)==0 {print $1}' /etc/shadow　　查看是否存在空口令用户 

```

3.查看分析任务计划

a) 通过crontabl –l 查看当前的任务计划有哪些，是否有后门木马程序启动相关信息；
b) 查看etc目录任务计划相关文件，ls /etc/cron*

4.查看linux 开机启动程序

a) 查看rc.local文件（/etc/init.d/rc.local /etc/rc.local）
b) ls –alt /etc/init.d/
c) chkconfig --list(需要安装，apt-get install -y chkconfig)

5.查看系统用户登录信息

a) 使用lastlog命令，系统中所有用户最近一次登录信息。

b) 使用lastb命令，用于显示用户错误的登录列表

c) 使用last命令，用于显示用户最近登录信息（数据源为/var/log/wtmp，var/log/btmp）
    utmp文件中保存的是当前正在本系统中的用户的信息。
   wtmp文件中保存的是登录过本系统的用户的信息。

​ /var/log/wtmp 文件结构和/var/run/utmp 文件结构一样，都是引用/usr/include/bits/utmp.h 中的struct utmp

6.系统环境分析
```
env  打印环境变量
echo $PATH 分析系统路径有无敏感可疑信息
cat /etc/profile /etc/bash.bashrc 分析这些文件中环境变量
```

7.ssh服务分析

a) 比对ssh的版本
```
    ssh -V
```
b) 查看ssh配置文件和/usr/sbin/sshd的时间
```
    stat /usr/sbin/sshd
```

c) 查看ssh相关目录有无可疑的公钥存在。
ls -l /root/.ssh/ /etc/ssh/

### 日志分析

```
日志文件
/var/log/message       包括整体系统信息
/var/log/auth.log        包含系统授权信息，包括用户登录和使用的权限机制等
/var/log/userlog         记录所有等级用户信息的日志。
/var/log/cron           记录crontab命令是否被正确的执行
/var/log/xferlog(vsftpd.log)记录Linux FTP日志
/var/log/lastlog         记录登录的用户，可以使用命令lastlog查看
/var/log/secure         记录大多数应用输入的账号与密码，登录成功与否
var/log/wtmp　　      记录登录系统成功的账户信息，等同于命令last
var/log/faillog　　      记录登录系统不成功的账号信息，一般会被黑客删除
```
登录日志可以关注Accepted、Failed password 、invalid特殊关键字
```
定位有多少IP在爆破主机的root帐号
grep "Failed password for root" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr | more
登录成功的IP有哪些
grep "Accepted " /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr | more  
tail -400f demo.log #监控最后400行日志文件的变化 等价与 tail -n 400 -f （-f参数是实时）  
less demo.log #查看日志文件，支持上下滚屏，查找功能  
uniq -c demo.log  #标记该行重复的数量，不重复值为1 
grep -c 'ERROR' demo.log   #输出文件demo.log中查找所有包行ERROR的行的数量
```
