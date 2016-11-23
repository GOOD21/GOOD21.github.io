---
layout:     post
title:      "Synology NAS同步Dropbox和GoogleDrive"
date:       2016-11-23
author:     "GOOD21"
header-img: "img/post-bg-js-module.jpg"
tags:
    - 生活
    - NAS
---

#### Cloud Sync
Synology DSM自带的**Cloud Sync**支持同步各种网盘到指定的文件夹。

其中**百度云**、**Onedrive**在国内同步没什么问题，但是**Dropbox**和**GoogleDrive**因为GFW的原因，只有**科学上网**才可以用。

#### Docker
Synolocy DSM里面的**Docker**简直屌爆了，有了它你能干的事情就多了。

你可以在注册表里搜`shadowsocks-privoxy` 选择`gd41340811/shadowsocks-privoxy`。

![1](/img/in-post/nas-docker-ss-cloudsync/1.png)

#### privoxy

因为群晖NAS**只支持http代理**，所以必须要用**privoxy**。

这是根据bluebu这哥们改写的，他写的是代理全部，所有的请求都走shadowsocks代理，但是其实我们只需要Dropbox和GoogleDrive走代理，所以在privoxy里配置改为如下：

```
# forward-socks5  / 127.0.0.1:7070  .  # 打开就是代理全部请求
forward          /    .
forward-socks5  .dropbox*.com 127.0.0.1:7070  . # 代理dropbox的请求
forward-socks5  .*google*.* 127.0.0.1:7070  . # 代理googledrive相关请求
```

这里关于dropbox有个地方比较坑，几乎网上的文章写的配置都是这样的：

```
forward-socks5 .dropbox.com 127.0.0.1:7070 .
forward .dropbox.com:443 .
```

这样的话，在CloudSync里`暂停同步`之后再`恢复同步`是好用的，但是后续的10s一次检查就一直显示`连接中`，根据抓包的请求发现：

```
connecting cfl.dropboxstatic.com:443
connecting notify.dropboxapi.com:443
```

这些请求根本没走代理，改成`.dropbox*.com`之后就好使了。

在github上有个**gfwlist2privoxy**的repo，可以把所有gfwlist转换成privoxy的actionfiles，这样就实现了PAC。（然而感觉在NAS上并没有什么卵用...）

#### 按需同步

在CloudSync的设置里可以调整**轮询期**，默认是10s。

我是拿来做备份的，不需要实时性，所以改成了3600s。

![2](/img/in-post/nas-docker-ss-cloudsync/2.png)


