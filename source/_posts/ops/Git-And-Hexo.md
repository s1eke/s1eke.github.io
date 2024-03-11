---
title: 用git服务器和hexo搭建博客
date: 2017-05-15 14:43:13
tags: 
    - Git
    - Hexo
    - Nginx
categories:
  - [运维]
---

## Git安装

  git 安装非常简单，几乎所有的发行版都能在官方源找到，直接用包管理器下载即可。由于我的 vps 使用的 debian ，所以：

	#apt install git 


## Git使用

1. Git仓库初始化

    git 的初始化仓库命令是 git init，git 服务器一般都会带上 –bare 参数，这样不会在服务器上生成工作目录，不过由于我的需求是通过 git 同步 hexo 站点，所以：

        #git init weibo

    当然，在此之前还要创建个 git 用户，最好把仓库放在git用户有权限的目录下或者用root创建目录，再 chown 给 git 用户。



2. Git同步

    我的需求是 git+nginx+hexo ，完成本地编写网页，再通过 git 来 push 到 nginx 目录下。所以上一步的初始化工作就默认是在 nginx 的目录下完成的。

    首先我们要修改服务器上的两个文件，一个是 .git/config ，因为默认的配置是不允许 push 操作的，这和我们的需求不同，需要添加如下代码：

        [receive]
        denyCurrentBranch = ignore

    解决了不能 push 的问题后，还要解决一个不能同步的问题，因为git服务器的主要作用还是共享数据，所以本地工作目录的文件在别的设备 push 上来后并不会即时同步，必须要手动输入 git reset –hard 才能同步，为了让 git 自动同步 push 内容到 work directory 这里就要用到 git 的 hooks 功能，也就是钩子。

    仓库的 .git 目录下有一个 hooks 目录，在仓库数据遭遇改变时，hooks下的脚本就会自动执行。所以我们要在这个目录下创建一个新的脚本：

        root@vultr ~www/html/weibo (git)-[master] # cat .git/hooks/post-receive 
        #!/bin/bash
        git --work-tree=/var/www/html/weibo checkout -f

    然后再确定此文件归属 git 用户，还要给予他可执行权限：

        # chown git:git post-receive
        # chmod +x post-receive

    OK了，现在要开始在本地搭建hexo站点，这个非常简单，暂不赘述。搭建完后，使用 hexo g 生成静态文件，再进入静态文件目录 clone 下服务端的仓库：

        # git clone git@sieke.lt:/var/www/html/weibo

    接下来就很简单了，把hexo生成的静态页面再 push 上远程仓库，hooks会自动让仓库的工作目录自动同步 push 内容，一个 git+nginx+hexo 的组合就这么完成了。
    最后顺便说一下 push 过程(ps.以readme.txt为例.)：

        # git add readme.txt
        # git commit -m "此处填写改动声明。"
        # git push origin master

## happy(´▽`ʃ♡ƪ)
