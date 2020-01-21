---
title: Nessus-安装离线激活并升级
date: 2018-05-13 15:29:12
updated: 2018-05-13 15:29:12
categories:
- 渗透测试
- 工具教程
tags: Nessus
---

nessus是一个漏洞扫描软件,可扫描web、主机等。其安装完成后会自动升级版本和插件库，在线升级会比较慢，可使用离线安装升级。
---
# 1.注册、下载、安装
注册Nessus家庭版激活码，获取软件下载链接
http://www.tenable.com/products/nessus-home
邮箱获取activation code，下载并安装，默认安装即可，安装到输入激活码时，按下面步骤在命令行中进行。

# 2.获取challenge code（机器码）
```
C:\Program Files\Tenable\Nessus>nessuscli fetch --challenge

Challenge code: d51cc6c47f41b8381d92b8261b50512b9974db8c

You can copy the challenge code above and paste it alongside your
Activation Code at:
https://plugins.nessus.org/v2/offline.php
```
# 3.获取nessus.license激活文件和all-2.0.tar.gz离线升级包

浏览器打开https://plugins.nessus.org/v2/offline.php，粘贴challenge code 和activation code（邮箱获取）到这里，提交跳转到一个结果页面获取到all-2.0.tar.gz离线升级包下载地址和nessus.license文件（最下面）。
# 4.开始激活
```
C:\Program Files\Tenable\Nessus>nessuscli fetch --register-offline nessus.licens
e
Your Activation Code has been registered properly - thank you.
```
# 5.开始升级
nessuscli update all-2.0.tar.gz
# 6.重启
nessusd restart

---
本文为原创：转载请注明:http://wiki.viewcn.cn/