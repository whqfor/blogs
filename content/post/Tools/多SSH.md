

---
title: "git 不同的邮箱配置不同的ssh"
date: 2019-04-25T15:05:23+08:00
lastmod: 2019-04-25T15:05:23+08:00
draft: false
tags: ["git", "SSH"]
categories: ["Tools"]
---

###### 背景：
1.提交代码到公司gerrit，需要使用公司邮箱aaa.@company.com 生成的SSH。
2.提交代码到github，想使用个人邮箱bbb.@126.com 生成的SSH。

环境：Mac系统

###### 配置ssh keys 
如果之前已经配置过其中一个，现在只需新配置另一个即可。
```
ssh-keygen -t rsa -f ~/.ssh/id_rsa[.别名] -C “邮箱地址“

示例：
ssh-keygen -t rsa -f ~/.ssh/id_rsa.github -C “bbb@126.com“
```
如果都没配置过，再创建执行另一个，别名和邮箱：
```
ssh-keygen -t rsa -f ~/.ssh/id_rsa.gerrit -C “aaa@company.com“
```
这样如果进入 .ssh 目录会看到有两套配置文件`id_rsa.github`、`id_rsa.github.pub`，`id_rsa.gerrit`、`id_rsa.gerrit.pub`。

###### 配置config

配置完ssh keys之后需要绑定一下对应关系，这个操作是在~/.ssh/config 目录下进行的。
如果是第一次配置本地应该没有这个文件。
```
cd .ssh
touch config
```
之后进行编辑即可，可以直接操作文本或者`vim config`进行编辑。
示例：
```
	# gerrit
	Host gerrit.company.com
	HostName gerrit.company.com
	Port 25638
	User aaa
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/id_rsa.gerrit
	
	
	# GitHub
	Host github.com
	HostName github.com
	User bbb
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/id_rsa
```
一般ssh config 只配置上面简单的几项即可。
如果想详细了解还有哪些参数参考[详细config参数](http://man.openbsd.org/ssh_config.5) ，
更多功能比如[ssh 在客户端如何强制使用密码验证方式登陆？](https://segmentfault.com/q/1010000000631942)。

之后将配置好的SSH key 添加到github 及公司的gerrit 即可。

在这个过程中你可能会使用到这些命令
```
显示/隐藏Mac隐藏文件命令如下(注意其中的空格并且区分大小写)：
显示Mac隐藏文件的命令：defaults write com.apple.finder AppleShowAllFiles -bool true
隐藏Mac隐藏文件的命令：defaults write com.apple.finder AppleShowAllFiles -bool false

终端查看ssk pub 
cat ~/.ssh/id_rsa.pub
将ssh key拷贝到剪切板： 
pbcopy < ~/.ssh/id_rsa.pub

```

###### 验证
最后验证一下是否配置好了:
```
验证github  
ssh -T git@github.com

localhost:.ssh whqfor$ ssh -T git@github.com
Enter passphrase for key '/Users/aaa/.ssh/id_rsa.github': 
Hi aaa! You've successfully authenticated, but GitHub does not provide shell access.
这就说明验证通过了
```

```
验证github  
ssh -T gerrit.compant.com

localhost:.ssh whqfor$ ssh -T gerrit.bbb.com
Enter passphrase for key '/Users/whqfor/.ssh/id_rsa.gerrit': 

  ****    Welcome to Gerrit Code Review    ****

  Hi bbb, you have successfully connected over SSH.

  Unfortunately, interactive shells are disabled.
  To clone a hosted Git repository, use:

  git clone ssh://bbb.company.com:25638/REPOSITORY_NAME.git

```


可以看下两个 验证方式有点小差别：`ssh -T git@github.com` 和 `ssh -T gerrit.compant.com`，具体形式没有找到说明，如果你知道，还请指导我，谢谢。


[可能还会遇到问题](https://stackoverflow.com/questions/40650926/git-push-error-does-not-match-your-user-account)
![多SSH.png](https://upload-images.jianshu.io/upload_images/273788-cac008f23cd35d98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我第一次配置多SSH的时候的确遇到这个问题，
解决方式好像是需要再配置一下邮箱，记不太清了，这个场景再次尝试没有复现，有遇到的话欢迎找我交流。

而配置邮箱你可能会需要
```
配置用户名
git config --global user.name "XXX" 

配置邮箱
git config --global user.email "XXX@XX.com"

取消用户名
git config --unset --global user.name

List all variables set in config file, along with their values.
git config --list
```
更多config配置参考[git-config](https://git-scm.com/docs/git-config/)


###### 参考
[Multiple GitHub Accounts & SSH Config](https://stackoverflow.com/questions/3225862/multiple-github-accounts-ssh-config)

[git 用不同的邮箱配置不同的ssh](https://www.cnblogs.com/tangranyang/p/5229300.html)

[Git配置多个SSH key](https://blog.csdn.net/hao495430759/article/details/80673568)

[Mac OS 配置多个ssh-key](https://blog.csdn.net/maoxinwen1/article/details/80269299)

[git push 问题：committer 'xxx (x)' does not match your user account](https://segmentfault.com/a/1190000008739604)


