---
title: github多账户设置
date: 2019-05-15 10:28:45
tags:
---

如果有两个github账户，要在windows下同时操作它们，和单账户的配置有些不同

假设
* 两个github的用户名分别为test1、test2
* 两个github账户绑定的邮箱分别为test1@example.com和test2@example.com
* github与windows下的用户名一致

## 创建ssh key
首先进入ssh目录
```cmd
cd ~/.ssh
```

为test1创建ssk key
```cmd
ssh-keygen -t rsa -b 4096 -C "test1@example.com"
```

输入test1保存的密钥文件名称
```cmd
Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]
# 这里输入 test1_id_rsa
```

输入密码
```cmd
Enter passphrase (empty for no passphrase): [Type a passphrase]
Enter same passphrase again: [Type passphrase again]
```
test2的ssk key创建步骤同上，保存的密钥文件为test2_id_rsa

## ssh key 添加到ssh-agent
ssh-agent在后台启动
```cmd
eval "$(ssh-agent -s)"
```

将两个密钥文件添加到ssh-agent
```cmd
ssh-add ~/.ssh/test1_id_rsa
ssh-add ~/.ssh/test2_id_rsa
```

## 配置文件config
配置或者创建并配置config文件(还是在~/.ssh/路径下)
```txt
Host test1.github.com
  HostName github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/test1_id_rsa

Host test2.github.com
  HostName github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/test2_id_rsa
```

## ssh key 添加到github
在test1 github页面的Settings-->SSH keys and GPG keys中，点击`New SSH key`，
将test1_id_rsa.pub文件中的所有内容复制粘贴过去

test2同上

## 测试
测试test1、test2
```cmd
ssh -T git@test1.github.com
ssh -T git@test2.github.com
```

## clone项目到本地
例如test1的远程仓库中有project1
```cmd
git clone git@test1.github.com:test1/project1.git
```
第一个test1是该账户在windows上配置的config文件中的HostName，后一个test1是github上的用户名
