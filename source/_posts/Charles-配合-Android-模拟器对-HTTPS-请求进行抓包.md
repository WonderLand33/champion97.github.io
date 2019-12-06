---
title: Charles 配合 Android 模拟器对 HTTPS 请求进行抓包
date: 2019-11-21 15:31:11
tags:
---






## Charles 配合 Android 模拟器对 HTTPS 请求进行抓包

### 要用到的东西：

 1.Charles
 2.Android 模拟器(网易mumu、夜神都可以，或者真机)
 3.要抓包的App(豆瓣app)



### 步骤：



1. 打开Charles，配置好HTTP代理

   ![MI8Axg.png](https://s2.ax1x.com/2019/11/21/MI8Axg.png)

   ![MI8DzD.png](https://s2.ax1x.com/2019/11/21/MI8DzD.png)





2. 打开安卓模拟器，设置->WLAN->长按->修改网络设置->手动设置代理

   ![MI07C9.png](https://s2.ax1x.com/2019/11/21/MI07C9.png)





3. 安装 SSL 根证书

   Charles ：

![MI8TyQ.png](https://s2.ax1x.com/2019/11/21/MI8TyQ.png)



![MIN7ad.png](https://s2.ax1x.com/2019/11/21/MIN7ad.png)

安卓模拟器通过浏览器访问 `chls.pro/ssl` 并下载：



![MIUFGq.png](https://s2.ax1x.com/2019/11/21/MIUFGq.png)



点击以安装证书。然后打开 设置->安全->信任的凭证会看到刚才安装好的ssl证书：

![MIUJsO.png](https://s2.ax1x.com/2019/11/21/MIUJsO.png)



4.在模拟器上打开需要抓包的APP，打开后观察Charles，这时候会有请求过来：





可以看到，请求是加密的，此时看不到，需要**开启 SSL Proxy**。



![MIdPg0.png](https://s2.ax1x.com/2019/11/21/MIdPg0.png)



开启 SSL Proxy 之后：

![MId1KK.png](https://s2.ax1x.com/2019/11/21/MId1KK.png)



### 总结

Charles、Fiddler 这类工具通过往系统导入根证书来实现 HTTPS 流量解密，充当中间人角色。要防范真正的 HTTPS 中间人攻击，网站方需要保管好自己的证书私钥和域名认证信息，为了防范不良 CA 非法向第三方签发自己的网站证书，还要尽可能启用 [Certificate Transparency](https://imququ.com/post/certificate-transparency.html)、[HTTP Public Key Pinning](https://imququ.com/post/http-public-key-pinning.html) 等策略；用户方不要随便信任来历不明的证书，更不要随意导入证书到根证书列表，还要养成经常检查常用网站证书链的习惯。

RSA 密钥交换没有前向安全性，这意味着一旦私钥泄漏，之前所有加密流量都可以解开。为此，网站方需要启用使用 ECDHE 作为密钥交换的 CipherSuite，或者直接使用 ECC 证书；用户方需要弃用不支持 ECDHE 的古董操作系统及浏览器。

对于浏览器而言，HTTPS 毫无秘密，通过浏览器生成的 `SSLKEYLOGFILE` 文件，Wireshark 可以轻松解密 HTTPS 流量。另外，如果浏览器被安装恶意扩展，即使访问安全的 HTTPS 网站，提交的数据一样可以被截获。这种客户端被攻击者控制引发的安全问题，无法通过 HTTPS 来解决。





参考：

https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%AE%89%E5%85%A8%E5%8D%8F%E8%AE%AE

https://imququ.com/post/how-to-decrypt-https.html



