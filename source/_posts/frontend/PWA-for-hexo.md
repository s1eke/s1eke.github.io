---
title: 让 Hexo 博客支持 PWA
date: 2018-04-04 23:59:49
tags: 
    - Node.js
    - PWA
categories:
  - [前端]
---


> Progressive Web Apps (PWAs) are web applications that are regular web pages or websites, but can appear to the user like traditional applications or native mobile applications. The application type attempts to combine features offered by most modern browsers with the benefits of a mobile experience.

--摘自维基百科

 

## 支持 HTTPS

支持 PWA 的第一步，就是全站 HTTPS。我的个人博客呢，是用 Hexo 生成静态页面，然后再通过 Nginx 进行代理，所以直接申请个免费的 SSL 证书就好了。（但是被图床的 HTTP 资源折腾了一下，=。=这个暂且不谈。）

## 配置 hexo-pwa 插件

Hexo 支持 PWA 非常方便，因为已经有了专门的模块 [hexo-pwa](https://github.com/lavas-project/hexo-pwa)，直接在博客目录用一行命令安装就行了：

```shell
$ npm install --save hexo-pwa
```

然后再配置站点的_config.yml文件，在最后添加 PWA 配置：

```yml
pwa:
  manifest:
    path: /manifest.json
    body:
      name: hexo
      short_name: hexo
      icons:
        - src: /images/android-chrome-192x192.png
          sizes: 192x192
          type: image/png
  serviceWorker:
    path: /sw.js
    preload:
      urls:
        - /*
      posts: 5
    opts:
      networkTimeoutSeconds: 5
    routes:
      - pattern: !!js/regexp /hm.baidu.com/
        strategy: networkOnly
      - pattern: !!js/regexp /.*\.(js|css|jpg|jpeg|png|gif)$/
        strategy: cacheFirst
      - pattern: !!js/regexp /\//
        strategy: networkFirst
  priority: 5
```

### 配置内容介绍
这里的配置中，第一段 manifest，里面的path就是你的 manifest.json 的位置；body 则是 manifest.json 的内容，可以写满，也可以为空。为空的话就会按照你的 path 设置去读取你的 manifest.json 文件内容。这个 manifest.json 也是 PWA 的关键内容，如果嫌配置麻烦的话，可以用 [App Manifest Generator](https://app-manifest.firebaseapp.com/?spm=a2c4e.11153940.blogcont441967.16.18ca18eaBudCNI) 这个网站一键生成。把生成完的配置文件丢到 /source 文件夹下就行了。

第二段的 serviceWorker 是 service worker 的配置文件，path 还是填写你的 sw.js 的位置。preload 则是填写你想要缓存的网址，设置为 /* 则是缓存所有页面，里面的 posts 是则是你想要缓存的页面数量上限。opts 和 routes 可以通过 [sw-toolbox](https://googlechromelabs.github.io/sw-toolbox/#main) 了解。

最后的 priority 是插件优先级的设置，相关内容可以点 [plugin priority](https://hexo.io/api/filter.html) 深入了解。


这里还有一个坑，就是在 Safari 如果添加到主页，他默认会选择你添加的那个页面的截图作为图标，而不是你设置的图标，所以这里需要单独为 iOS 用户生成一个图标: source/apple-touch-icon.png(ps.iOS 默认用黑色填充透明图层，所以建议不要选择又透明图层的图片作为图标，或者手动用白色填充下)。更多配置参考 [Configuring Web Applications](https://developer.apple.com/library/content/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html)。
iOS上的坑还有很多，诸如状态栏外观、链接到外部程序等，这也是#TODO的一部分，日后再谈。
作为示例，这是我的 [manifest.json](https://cfp.vim-cn.com/cbfbq/json) 。

当然了，这只是初步支持 PWA，日后支持推送、弹窗等功能后我再继续更新吧。（逃

学习资料：[百度 LAVAS](https://lavas.baidu.com/pwa) 、[Progressive Web Apps (PWA) 中文版](http://sangka-z.com/PWA-Book-CN/)
