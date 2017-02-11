---
layout:     post
title:      "WoSign 和 StartCom 证书被禁之后"
date:       2017-02-11
author:     "GOOD21"
header-img: "img/post-bg-js-module.jpg"
tags:
    - 生活
    - NAS
---

最近发现家里的NAS 在Chrome打不开了，一直报`NET::ERR_CERT_AUTHORITY_INVALID`，但是safari 能打开。

一开始认为是chrome的bug，还特意去google查了一下，并没有什么结论，感觉像是网络抽风，没太在意。

直到周六放假，强迫症的我实在是忍不了了，仔细查了下，原来是 Chrome 在2016年10月份正式把WoSign 和 StartCom 的证书置为不信任状态，在Chrome 56 版本之后绑定WoSign 和 StartCom 证书的网站将无法打开。（详见：[Distrusting WoSign and StartCom Certificates](https://security.googleblog.com/2016/10/distrusting-wosign-and-startcom.html)）

这就很尴尬了，我用的正好是StartCom的免费证书，而且刚升级的Chrome 56...所以开始了纠结的换证书之旅...

首选的是`Let's Encrypt`，Synology里面有集成的工具，但是因为需要对外开80端口，所以基本上验证不通过。外加这个证书的有效期是3个月，虽然可以启个计划任务3个月更新一次，但是想想还是太恶心了，而且越来越不敢用免费的东西了...

然后在各种商家选，最后选择了[https://www.ssls.com/](https://www.ssls.com/) 的Positive SSL，$4.99/YR，一口气整了3年的。

有意思的是这家验证域名的owner时，只有发送域名邮件和http访问页面的方式，而我既没有企业邮箱记录，也没打算绑个服务器，后来跟客服妹子聊，可以通过CNAME验证，这就省了不少事儿。

搞定之后又可以愉快的登录NAS了，神清气爽！

元宵节快乐！喵~~

![miaomiao.jpeg](/img/in-post/update-ssl/miaomiao.jpeg)

