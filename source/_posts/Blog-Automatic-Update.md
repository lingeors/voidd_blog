---
title: 使用 git hooks 实现自动更新
date: 2017-07-12 22:43:57
tags:
---

> 我的博客用的是 hexo，因此博客所有文件分为两个部分，第一部分由markdown源文件以及相关的配置文件组成，我将它们备份到了github上；第二部分为生成的静态文件，这部分自然是要部署到业务服务器上。一开始我是通过 ftp 直接上传文件，后来找了个时间使用 git hooks 实现了文件的自动化更新。整个过程不算复杂，但其中还是有挺多坑的（花了我整整一个下午），记录一下。

<!-- more -->

最终的实现效果为：**本地生成静态文件，将静态文件 push 到远程服务器，即可在浏览器访问到更新后的文件。**

实现方案如下：
![](/images/Blog-Automatic-Update/struct.jpg)

步骤一中我们将静态文件push到远程git服务器上；
步骤二通过 git 的钩子机制更新业务服务器上的代码文件
经历了上述步骤后，我们可以通过网络访问到最新的文件

关于上述步骤我们需要注意两点：
1. 关于步骤一，我们实际可以选择 github或者oschina的内置钩子机制，参考 [码云平台帮助文档_V1.2][1]，但这里我们选择自建git服务器，相对来说这种方法更加灵活。
2. 步骤一的git服务器和步骤二的业务服务器我们部署在同一台服务器上。


## git服务器搭建
> 注意本文使用的服务器系统为 centos 7

首先搭建自己的git服务器，本地push的文件会被提交到该服务器，它的作用类似与码云和github，区别在于我们可以更加方便地利用git的钩子机制实现相关功能。
搭建git服务器可以参考 [搭建Git服务器 - 廖雪峰的官方网站][2] 或 [Git - 协议][3]。可能是版本或系统相关原因，这些博客或文档都忽略了相关问题。所以这里我把整个过程重新叙述一遍。

前几个步骤直接查看 [搭建Git服务器 - 廖雪峰的官方网站][2] ，直到步骤5。
到这里我们就拥有了一个git服务器仓库。
接下来步骤如下：
步骤一：查看 `/etc/ssh/sshd_config` 文件，找到
```bash
RSAAuthentication yes        # 启用 RSA 认证，默认为yes
PubkeyAuthentication yes     # 启用公钥认证，默认为yes
```
将前面的注释去掉，接着，重启ssh
```bash
sudo systemctl restart sshd.service # 这里是 centos7 系统命令，其它系统请查阅相关文档
```
步骤二：修改权限。执行如下命令
```bash
su git
cd /home/git && chmod 700 .ssh
chmod 600 .ssh/authorized_keys
```
注意设置上述文件或文件夹的权限，否则将会失败

经过上述步骤后，我们就可以在使用本机连接远程git服务器了。首先我们将本机的 `.ssh/id_rsa.pub` 中的内容添加到 `authorized_keys` 文件中。然后通过 `git remote add origin git@server:/srv/sample.git` 添加远程服务器。接着文件并通过 `git add, git commit, git push` 等操作就可以直接将本地git内容推送到远程git服务器。注意这时候 push 是不需要填写密码的，这是基于 ssh公钥认证机制（我们前面已经将公钥添加到了git服务器的authorized_keys）。这个过程和推送到github上不需要密码是一致的。

## git hooks编写
关于 `git hooks` 的具体类型我们可以参考这里 [Git - Git 钩子][4]。简单来说，钩子的作用类似于回调函数，当某些事件发生时，就会触发对应的钩子，上述文档即定义了相关的钩子事件（类型）。**我们要做的就是编写对应事件的钩子脚本，当事件发生时，这些脚本被执行。**
这里我们使用的是 `post-receive` 这个钩子，因此我们要做的就是在git服务器仓库的 `.git/ hooks` 中创建 `post-receive` 文件（不需要后缀），并用 `shell, python, ruby, php` 等脚本语言编写脚本，这里我们使用shell

脚本大概如下：
```bash
#!/bin/bash

# 执行 shell 命令
unset GIT_DIR
# 具体代码目录
cd /home/git/source_code
git pull origin master
```
编辑好该文件后，注意添加执行权限 `chown +x post-receive`
该脚本十分简单，首先切换到我们想要更新的具体目录，接着执行 `git pull` 完成更新。
这个过程需要注意几点：
- `post-receive` 需要先设置为可执行 `chmod u+x post-recieve`
- 一定要先执行 `unset GIT_DIR` 否则产生如下错误 `fatal: Not a git repository: '.'`（真是个大坑，找了好久）。具体参考这里 [Git Loves the Environment][5]。在 post-receive hooks 中，·`$GET_DIR` 被设置为 `.`，因为无论我们切换到其他哪个文件夹，都无法看到 `.git` 文件。也就会提示所在目录没有初始化
- `source_code` 需要先初始化 `git init` 并通过 `git remote add...` 添加git服务器
- 完成上述步骤后可能还会出现很多问题，很大可能是文件权限导致的。有两种解决方法，第一是更换 source_code 的所有者，即 `chown -R git:git source_code`；第二种方法是赋予git执行该文件夹的权限，具体操作可自行google

最终执行结果如下：
![](/images/Blog-Automatic-Update/push-result.jpg)

## 最后
整个部署过程最大的问题应该在一些小细节上，比如 `ssh`服务配置修改，`.ssh` 等文件和文件夹的权限修改等，其它步骤比较清晰并有很多文档，应该是比较简单的。
最后，以后生成静态文件后只要 `git push` 即可将最新文件部署到我们的业务服务器上，这种感觉真的很棒

## 参考
[码云平台帮助文档_V1.2][1]
[搭建Git服务器 - 廖雪峰的官方网站][2]
[Git - 协议][3]
[Git - Git 钩子][4]

[1]: http://git.mydoc.io/?t=153693
[2]: http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137583770360579bc4b458f044ce7afed3df579123eca000
[3]: https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%8D%8F%E8%AE%AE
[4]: https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90
[5]: https://git-scm.com/blog/2010/04/11/environment.html
