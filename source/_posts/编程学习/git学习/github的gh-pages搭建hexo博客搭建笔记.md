---
title: github和hexo博客搭建笔记
date: 2018-05-13 15:20:11
updated: 2023-03-24 16:05:11
categories:
- 编程学习
- git学习
tags: [git,hexo]
---

# 安装node

```
1.下载安装node版本管理工具
npm install -g n --registry https://registry.npm.taobao.org
2.下载指定版本(后面跟版本号) //hexo版本依赖：https://hexo.io/zh-cn/docs/#%E5%AE%89%E8%A3%85-Hexo
sudo n 10.5.0
3.下载最新版本
sudo n latest
sudo n stable
4.显示已安装哪些版本
n ls
5.切换使用版本(后面跟版本号)
sudo n 10.5.0
也可以输入
sudo n
查看已安装版本，上下切换使用哪个版本
6.删除指定版本
sudo n rm 10.5.0

7.其他-升级npm
sudo npm install -g npm
```

# 安装hexo

```
1.首次安装
sudo npm i hexo-cli -g
hexo init                           //首次安装,迁移不要使用
npm install hexo-deployer-git --save

2.升级
sudo npm i hexo-cli -g --force
//以下命令分别执行即可
sudo npm install -g npm-check     //安装npm-check
sudo npm-check                    //查看系统插件是否需要升级
sudo npm install -g npm-upgrade   //安装npm-upgrade
sudo npm-upgrade        //更新package.json
//在执行npm-upgrade命令后会要求输入yes或者no，直接输入Yes或Y即可
sudo npm update -g      //更新全局插件
sudo npm update --save  //更新系统插件
```
# 修改配置文件，发布到gh-pages分支

_config.yml
```
deploy:
  type: git
  repo: git@github.com:Chuanwei/Chuanwei-wiki.git
  branch: gh-pages
```
# 配置ssh
```

git>ssh-keygen -t rsa -C "xxxxx邮箱@qq.com"
Linux 系统：~/.ssh
Mac 系统：~/.ssh
Windows ：C:\Users\username\.ssh
最后把公钥id_rsa.pub的内容添加到 GitHub
```

# 配置github

```
新建一个项目Chuanwei-wiki

创建gh-pages分支
git branch  gh-pages
git checkout gh-pages
git push origin gh-pages

删除分支
git branch -d <BranchName>
git push origin --delete <BranchName>

打开Chuanwei-wiki项目的配置settings界面
https://github.com/Chuanwei/Chuanwei-wiki/settings

Source 修改为gh-pages

Custom domain域名设置为自己的域名：wiki.viewcn.cn

添加CNAME文件到gh-pages分支目录中，文件名称：CNAME，内容：wiki.viewcn.cn

https://github.com/Chuanwei/Chuanwei-wiki/blob/gh-pages/CNAME

```

# 修改dns映射

我的域名是在dnspod，方法都差不多。
新建一条cname记录：
主机记录：wiki   记录值：Chuanwei.github.io.
Chuanwei,是我的github名称,使用ping能通Chuanwei.github.io，说明是正常的。
此时可以使用wiki.viewcn.cn访问博客了，还没生成静态文件，现在前台gh-pages分支是空的，无法访问。


# 日常发布文件命令
下面将在maste分支上执行文章发布，会把网站静态文件发布到gh-pages分支，打开网站即显示的是gh-pages分支内容。
```
//建议第一步拉取最新的github上的master
git pull

//电脑上写好的文章md放入文件夹
source\_posts
//开始发布文章
hexo clean
hexo generate
hexo deploy
hexo g -d

//本地master提交到github
git add .
git commit -m "添加文章"
git push origin master

//到此文章发布完成

//切换到gh-pages分支，pull拉取远程gh-pages，可本地查看前台。
git checkout gh-pages
//pull github上最新的gh-pages。
git pull origin/gh-pages
//如果报错则强制覆盖本地gh-pages
git fetch --all
git reset --hard origin/gh-pages 
git pull
```

# 多终端发布

```
//每次发布完文章，push一下，方便在家里电脑pull，继续工作。
git add .
git commit -m "添加文章"
git push origin master
//在家电脑写文章
//首次安装一下
npm i -g hexo
npm install hexo-deployer-git --save

//pull github上最新的master。
git pull origin/master
//如果报错则强制覆盖本地master
git fetch --all
git reset --hard origin/master 
git pull origin/master
```
# 设置国内npm源,解决无法安装hexo
```
//临时的
npm i -g hexo --registry https://registry.npm.taobao.org
//持久的
npm config set registry https://registry.npm.taobao.org 
```

至此master分支作为后台文章发布和hexo配置，gh-pages分支作为前台页面。

---
本文为原创：转载请注明:http://wiki.viewcn.cn/