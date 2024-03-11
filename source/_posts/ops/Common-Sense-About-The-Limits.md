---
layout: post
title: 关于 Limits 的一些常识
date: 2019-09-23 01:09:42
tags: 
    - Limits
    - Linux
categories:
  - [运维]
---

## 前言

最近部署的一个项目，启动十几个小时后就报错了，看了下日志输出是 `Too many open files` 。放到搜索引擎查了下，了解到是系统默认进程最大打开文件描述符数太小导致的，所以解决问题后顺便来记录一下。~~（其实是为了水一篇博客）~~

## 关于用户的限制

`/etc/security/limits.conf` 文件是用来限制用户的资源使用，防止系统被fork炸弹占用所有资源的有效方法，具体配置信息可以在 [limit.conf (Arch manual pages)](https://jlk.fjfi.cvut.cz/arch/manpages/man/limits.conf.5) 里详细了解，可配置项很多。我们日常要用到的主要是其中的两项：`nproc` 和 `nofile`。

`nproc` 是用户最大可打开进程数。用 `# ps -u <user>|grep -v PID|wc -l` 可以查看用户当然打开进程数。（ps，grep，wc这三个进程也包括在内）下面提供两个`nproc`的修改示例。

```shell
*           hard    nproc      2048  # *匹配所有用户；%可以匹配组
root        hard    nproc      65536 # hard是硬限制，是系统不允许用户启动进程超过的上限；soft是软限制，是由用户限制进程不得超过的上限
```

`nofile` 是进程最大可打开文件描述符数。用`ulimit -n`可以查看当前用户的限制，用 `$ cat /proc/<PID>/limits` 可以该 PID 的所有资源限制。下面提供两个`nofile`的修改示例。

```shell
*           hard    nofile     8192 
root        hard    nofile     unlimited # unlimited是不做限制
```

## 关于系统的限制

系统最大可打开的文件描述符数通过 `$ cat /proc/sys/fs/file-max` 查看，当前系统打开文件描述符总数可以通过 `# cat /proc/sys/fs/file-nr` 查看，其中第一个数表示当前系统正在使用的文件描述符数，第二个数为目前不再使用的，第三个数则等于`/proc/sys/fs/file-max`。而如果要修改系统文件描述符限制则需要更改 `/etc/sysctl.conf` 文件，例如：
```shell
# echo "fs.file-max = 1000000" >> /etc/sysctl.conf
```
另外，file-max是限制系统内核可分配的最大文件数，而单个进程最大可分配的文件数则是用 `$ cat /proc/sys/fs/nr_open` 查看，修改方式则较为类似：
```shell 
# echo "fs.nr_open = 1000000" >> /etc/sysctl.conf # nr_open不可大于file-max
```
## 结语

简单的介绍了一下关于设备资源的限制的常识，可以发现其中很多都会牵扯到 `/proc` 这个目录，这是一个让你能和内核内部数据结构进行交互，获取相关进程有用信息的目录，也是一个独立的文件系统，下次我们就来聊聊 `/proc` 吧。



