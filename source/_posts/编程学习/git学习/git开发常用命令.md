---
title: git开发常用命令
date: 2023-03-24 16:14:06
updated: 2023-03-24 16:14:06
categories:
- 编程学习
- git学习
tags: [git,code]
---


# 配置ssh密钥
```
git>ssh-keygen -t rsa -C "xxxxx邮箱@qq.com"
Linux 系统：~/.ssh
Mac 系统：~/.ssh
Windows ：C:\Users\username\.ssh
最后把公钥id_rsa.pub的内容添加到 GitHub
```


# 常用命令

```
创建分支
git branch dev
git checkout dev

本地分支推送到远程
git push origin dev

删除分支
git branch -d <BranchName>

删除远程分支
git push origin --delete <BranchName>

查看远程分支
git remote -v

添加远程仓库（自己的远程仓库或上游仓库）
git remote add upstream  https://gitee.com/liqianglog/django-vue-admin.git
删除远程仓库
git remote remove upstream

将本地更新推送到远程分支
git add .
git commit -m "update"
git pull origin master
git pull origin master --rebase  //优化日志

# 关联 远程分支 Branch 'gh-pages' set up to track remote branch 'gh-pages' from 'origin'.
git pull简化拉取

git branch --set-upstream-to=origin/gh-pages gh-pages
```

# git pull冲突解决

```
1.git pull有冲突的解决办法
git fetch --all                  
git reset --hard origin/master  强制合并
```

# 删除误上传的敏感文件

```
git rm -r --cached .idea/
git filter-branch --tree-filter 'rm -rf mapp/plugins/gsil/config.gsil.cfg' HEAD
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch mapp/plugins/gsil/config.gsil.cfg' --prune-empty --tag-name-filter cat -- --all
git push origin --force --all
git push origin --force --tags
git push origin webm2.0:webm2.0 --force --all
git push origin webm2.0:webm2.0 --force --tags
git push origin webm2.0:master --force --all
git push origin webm2.0:master --force --tags
```

---
本文为原创：转载请注明:http://wiki.viewcn.cn/
