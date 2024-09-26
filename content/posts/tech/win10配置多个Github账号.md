---
title: "Win10配置多个Github账号"
date: 2024-09-25T14:11:40+08:00
lastmod: 2024-09-25T14:11:40+08:00
author: ["Lorenwe"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
reward: true # 打赏
mermaid: false #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "./posts/tech/win10配置多个Github账号/git-github.webp" #图片路径例如：posts/tech/123/123.png
    zoom: 50% # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---


### 一、为什么要配置多个Git账号

文章写于2021年的最后一天，最近闲来无事，想搞个导航网站，至于为什么要自己搞，纯属兴趣，既然是兴趣，就想着不要投入太大，整个省心省力的方案，github pages走起，问题就来了，一个github账号只能CANAME一个域名，我刚好挂了一个博客，其他的仓库要pages就只能在我的博客域名后面跟上仓库名这样访问，这很不程序员，所以我就又注册了一个github账号，现在在我的win10电脑上就要使用两个github账户了，当我在我的电脑上给新注册的仓库推送代码时就会出现当前用户没有权限访问，至此，就只能想办法配置好多个用户访问了，其实实际工作中也会遇见这个问题，比如自己肯定有github账户，开发公司项目的时候可能会有公司的gitlab账户或者其他代码仓库的账户，如果是使用http方式拉取代码还好，账户密码一输，记住密码也就没有后续问题了，但如果是使用ssh拉取代码就懵逼了，今天就是来解决这个问题的

### 二、初始化Git配置

默认我们都是已经使用了很久git，肯定做过很多设置了，为解决上面这个问题呢主要就是让git不再以一个用户身份对仓库进行更改，而是让它自动的选择对应的用户身份去操作代码仓库，只要用户身份和仓库对应上了，就没有权限问题啦，那就开干吧！

##### 1.去除全局用户配置

```shell
# 设置全局用户名
git config --golbal user.name "XXX"
# 设置全局邮箱
git config --golbal user.email "xxx@aa.com"
# 查看全局用户名
git config --global user.name
# 查看全局邮箱
git config --global user.email
# 查看全局密码
git config --global user.password
```

移除全局配置的用户信息

```shell
# 移除全局配置账户
git config --global --unset user.name
# 移除全局配置邮箱
git config --global --unset user.email
# 移除全局密码
git config --global --unset user.password
```

如果上述命令执行有问题的话其实还可以自行更改git配置文件来实现

如：找到`C:\Users\Administrator\.gitconfig`文件，用文本编辑器打开找到`[user]`这一栏把所有的用户及邮箱信息删掉，就像这样：

```ini
[user]
[http]
	sslVerify = false
```

接下来运行命令来检查一下配置是否成功 `git config --list`

```shell
$ git config --list
core.symlinks=false
core.autocrlf=true
core.fscache=true
color.diff=auto
color.status=auto
color.branch=auto
color.interactive=true
help.format=html
rebase.autosquash=true
http.sslbackend=openssl
http.sslcainfo=C:/Program Files/Git/mingw64/ssl/certs/ca-bundle.crt
credential.helper=manager
http.sslverify=false
```

没有看到用户信息就算完成了

##### 2.生成密钥对

因为是要通过ssh来推拉仓库，故我们要先生成ssh的key，生成的命令也很简单 

```shell
ssh-keygen -t rsa -C "lorenwe@163.com"
```

在这先声明一下自己的用户体系，以后后面搞混乱了，我原来自有一个github账户名是lorenwe，现在又注册了一个名为lorenwe2的账户（取名是个麻烦事），邮箱呢都是用的163的，所以两个账户的邮箱是lorenwe@163.com和lorenwe2@163.com，接下来就是不同的操作来啦

注意上面那条命令 `ssh-keygen -t rsa -C "lorenwe@163.com"` 执行完后会在用户目录也就是`C:\Users\Administrator` 目录的 `.ssh` 目录里面生成两个文件名为 id_rsa 和 id_rsa.pub 的两个文件，如果在执行命令前就有的话可能你之前已经配置了ssh key，如果知道key对应配置哪个网站的话，可以删除从新配置，如果不支持重新配置的话就不建议删除了，

可以使用这个命令生成指定文件名的密钥对 

```shell
ssh-keygen -t rsa -C "lorenwe@163.com" -f id_rsa_lorenwe
```

现在我们不指定，也就是说生成了上面所说的两个文件，然后对这两个文件进行重命名，为了好区分我就直接在后面加上用户名了，现在文件结构如下：

![ssh_lorenwe.png](/posts/tech/win10配置多个Github账号/ssh_lorenwe.png)

接下来就是为第二个用户生成 ssh key，重复以上操作即可

```shell
Administrator@PC-20210304RNJZ MINGW64 ~/.ssh
$ ssh-keygen -t rsa -C "lorenwe2@163.com" -f id_rsa_lorenwe2
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_rsa_lorenwe2.
Your public key has been saved in id_rsa_lorenwe2.pub.
The key fingerprint is:
SHA256:DwTCt/KL3YvvLEA9AFj/8BCGyRj/00EJLM4KXe+TCZk lorenwe2@163.com
The key's randomart image is:
+---[RSA 2048]----+
|.*o*=.o.         |
|o.=o=+o.         |
| +.o+B...        |
|. +.E*=o         |
|.. .o=++S        |
|.   ..*  o       |
|     + +  .      |
|    . +o.        |
|      .+=.       |
+----[SHA256]-----+
```

注意使用 `-f` 指定生成文件名时需要让命令行处于在 .ssh 目录下，因为指定文件名后生成的文件就处于命令行所在的文件夹，在 .ssh 目录下省去了再去移动文件这一步，现在我们的文件目录结构为：

![ssh_lorenwe2.png](/posts/tech/win10配置多个Github账号/ssh_lorenwe2.png)

至此准备工作完成了。

### 三、配置Github的 SSH

1. 用户 lorenwe 登录 GitHub，进入【Settings】-【SSH and GPG keys】，如下截图：

![github_step1.png](/posts/tech/win10配置多个Github账号/github_step1.png)

2. 点击【New SSH key】按钮，进入新建SSH key页面，复制 id_rsa_lorenwe.pub 文件内容粘贴到大框框中，如下配图：

![github_step2.png](/posts/tech/win10配置多个Github账号/github_step2.png)

   Title随便取一个，我这取的是邮箱，这样配置完了就知道我这个密钥是给哪个用户配置的

3. 打开另外一个浏览器登入lorenwe2 账户重复一遍上面操作即可，至此GitHub配置完成了，同样的道理我们也可以配置其他站点的 ssh key，如 gitee 等

### 四、测试配置是否成功

可以执行下面的命令来测试密钥配置，需要注意的是测试命令需要自己指定要使用的是哪个key，因为在.ssh目录下面有两个key，默认是使用id_rsa来验证的

```shell
ssh -T git@github.com -i ~/.ssh/id_rsa_lorenwe
Hi lorenwe! You've successfully authenticated, but GitHub does not provide shell access.
ssh -T git@github.com -i ~/.ssh/id_rsa_lorenwe2
Hi lorenwe2! You've successfully authenticated, but GitHub does not provide shell access.
```

看到 successfully 就说明成功了

### 五、Git配置不同站点使用不同用户

还是在.ssh目录下，创建一个文件名为 config 的文件，注意不带后缀，不是config.txt和config.ini，不带后缀就叫config，编辑器打开然后输入以下内容：

```ini
# 配置user1
Host lorenwe.github.com
HostName github.com
IdentityFile C:\\Users\\Administrator\\.ssh\\id_rsa_lorenwe
PreferredAuthentications publickey
User lorenwe

# 配置user2
Host lorenwe2.github.com
HostName github.com
IdentityFile C:\\Users\\Administrator\\.ssh\\id_rsa_lorenwe2
PreferredAuthentications publickey
User lorenwe2
```

假如你还要配置公司内部的gitlab用户，那么可以按照下面的配置来修改

```ini
# 配置user3
Host 192.168.12.5
Hostname 192.168.12.5
PreferredAuthentications publickey
IdentityFile C:\\Users\\Administrator\\.ssh\\id_rsa_gitlab
User gitlab_name
```

最后.ssh目录的文件结构如下图：

![ssh_list.png](/posts/tech/win10配置多个Github账号/ssh_list.png)

### 六、使用需要注意的地方

在github上创建好一个仓库，创建好后如下：

![github6.png](/posts/tech/win10配置多个Github账号/github6.png)

注意到 仓库 clone 路径 ssh的方式 

```ini
git@github.com:lorenwe2/lorenwe2.git
```

使用以下命令clone会发现出错了，具体如下

```shell
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test
$ git clone git@github.com:lorenwe2/lorenwe2.git
Cloning into 'lorenwe2'...
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.
```

原因就是之前配置的config中是以下配置

```ini
# 配置user1
Host lorenwe.github.com
HostName github.com
IdentityFile C:\\Users\\Administrator\\.ssh\\id_rsa_lorenwe
PreferredAuthentications publickey
User lorenwe

# 配置user2
Host lorenwe2.github.com
HostName github.com
IdentityFile C:\\Users\\Administrator\\.ssh\\id_rsa_lorenwe2
PreferredAuthentications publickey
User lorenwe2
```

两个用户的 HostName 都一样，git就从你给的clone链接里面拿到的host name也是github.com，这时候它就不知道要用哪个用户来clone了，解决办法很简单，把host再具体一点，如：

```shell
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test
$ git clone git@lorenwe2.github.com:lorenwe2/lorenwe2.git
Cloning into 'lorenwe2'...
warning: You appear to have cloned an empty repository.
```

上面就clone 成功了，只是clone了一个空仓库，我们只是在github.com前面加上了用户名，那就刚好对应上了上面config配置的Host，git 就知道要用哪个用户去clone仓库了，配置里面还定义的key的路径，那key也正确，所以就正常clone下来了，再测试一下推送功能吧

```shell
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test
$ cd lorenwe2/
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git status
On branch master
No commits yet
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        test.txt
nothing added to commit but untracked files present (use "git add" to track)
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git add -A
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git commit -m "添加文件"
*** Please tell me who you are.
Run
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
to set your account's default identity.
Omit --global to set the identity only in this repository.
fatal: unable to auto-detect email address (got 'Administrator@PC-20210304RNJZ.(none)')
```

可以发现命令执行到git commit时候就出错了，给的提示信息也很明显，没有设置全局用户名，不知道以什么用户身份去提交代码，我们不要按照他的说明来，他是要设置全局用户，但我们电脑是要使用多个用户的，所以设置全局用户显然不合适，可以使用以下命令为当前项目设置用户及邮箱

```shell
git config user.name "lorenwe2"
git config user.email "lorenwe2@163.com"
```

```shell
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git config user.name "lorenwe2"
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git config user.email "lorenwe2@163.com"
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git commit -m "添加文件"
[master (root-commit) 7bf22a5] 添加文件
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test.txt
Administrator@PC-20210304RNJZ MINGW64 ~/Desktop/test/lorenwe2 (master)
$ git push
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 214 bytes | 214.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To lorenwe2.github.com:lorenwe2/lorenwe2.git
 * [new branch]      master -> master
```

从上面可以看出，推送一切也都是ok的，假如不是新的仓库，而且原先还是使用HTTPS通信，则可以修改远程仓库地址

```shell
git remote rm origin
git remote add origin git@lorenwe2.github.com:xxx/xxxxx.git
```

再为这个仓库配置用户名和邮箱，就也能对旧仓库修改推送方式，如不知道旧仓库是如何配置的，可以打开仓库目录里的.git文件夹找到config文件查看即可，如图：

![lorenwe2_config.png](/posts/tech/win10配置多个Github账号/lorenwe2_config.png)

打开后可看到如下类似内容

```ini
[core]
	repositoryformatversion = 0
	filemode = false
	bare = false
	logallrefupdates = true
	symlinks = false
	ignorecase = true
[remote "origin"]
	url = git@lorenwe2.github.com:lorenwe2/lorenwe2.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
[user]
	name = lorenwe2
	email = lorenwe2@163.com
```

至此 win10 配置多个git账户算是完成啦
