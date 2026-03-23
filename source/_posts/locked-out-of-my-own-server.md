---
title: 记一次被自己服务器关在门外的经历
date: 2026-03-23 00:00:00
tags: server-admin
---

![IMG_8335.jpg](https://i.see.you/2026/03/23/v7Nv/IMG_8335.jpg)

在台北买了一台 NVIDIA DGX Spark，回到酒店配置时，因为 SSH Key 导入失败又顺手关掉了密码认证，把自己彻底锁在了门外。最终我靠着之前装好的 Cloudflared 隧道，一路过关斩将——绕过损坏的缓存装上 JupyterLab、用 Host Header Override 解决 403——才拿到 Terminal 修复了 SSH 配置，有惊无险地把自己救了回来。

<!-- more -->

在台北买了一台 NVIDIA DGX Spark，回到酒店正打算愉快地开箱配置。

首次设置时，DGX 会暴露一个 Wi-Fi 热点，连上之后通过 Web 面板就能配置网络。


![1.jpg](https://i.see.you/2026/03/23/y3Ko/1.jpg)

这里埋下了第一个坑——酒店的 Wi-Fi 需要通过 Captive Portal 页面认证，而 DGX 连上酒店 Wi-Fi 后无法弹出认证页面，导致网络不通、后续配置全部卡住。最终只好用手机开热点来完成初始化。


系统装好后，SSH 连上去，习惯性地导入 SSH Key。但不知为何导入失败了——我也没仔细核对，顺手就把 `PasswordAuthentication` 关了，然后重启了 sshd。好在 sshd 重启不会断开已有的连接，当时并没有意识到问题。

![2.jpg](https://i.see.you/2026/03/23/baC1/2.jpg)

接着又装了 Cloudflared，把机器接入自己的 ZeroTrust 网络。装完出去吃了个饭，回来发现 SSH 连接已经超时断开。尝试重连，反复提示 `Permission denied (publickey)`——这才意识到，SSH Key 根本没导入成功，而密码认证也被我亲手关掉了。

![pasted-image-1774289970360.webp](https://i.see.you/2026/03/23/A9qw/pasted-image-1774289970360.webp)

**我把自己锁在了门外。**

脑子开始飞速运转。查了一圈文档，发现 DGX 提供了一个 Web Dashboard，但只监听在 `localhost` 上，外部无法直接访问。突然想起来——之前装的 Cloudflared 还在跑！

![3.png](https://i.see.you/2026/03/23/m7Du/3.png)

赶紧进 ZeroTrust 后台，把端口 11000 映射出来，成功打开了 Dashboard。惊喜地发现上面可以安装 JupyterLab——只要能拿到一个 Terminal，就能修复 SSH 配置。

![4.png](https://i.see.you/2026/03/23/ubO1/4.png)

然而事情没那么顺利。大概是酒店网络不太稳定，在下载 `torch-2.8.0+cu129` 这个巨型 wheel 包时超时了，而且后续重试也反复报同样的错：

```
pip._vendor.urllib3.exceptions.ReadTimeoutError: HTTPSConnectionPool(host='download.pytorch.org', port=443): Read timed out.
```

这时已经在做最坏的打算了——重装系统。但手边既没有 U 盘，也没有键盘鼠标。找酒店借了一根 HDMI 线，接上电视后发现 DGX 只有开机画面有 HDMI 输出，进入系统之后屏幕就黑了。此路不通。

于是回到 DGX Dashboard 上继续想办法。发现点击启动 JupyterLab 时有一个 Working Directory 参数，虽然前端界面上改不了，但抓包发现 POST 请求里是可以传这个参数的。

![pasted-image-1774289697488.webp](https://i.see.you/2026/03/23/3jRf/pasted-image-1774289697488.webp)

尝试把路径改成 `/home/satori/jupyterlab/tmp` 居然成功绕过了之前损坏的缓存，在下载了114514个依赖没有失败之后，终于终于成功装上了 JupyterLab！

但故事还没完。JupyterLab 同样只监听在 `localhost:11002`，用 Cloudflared 转发之后打开页面，又遇到了 `403 Forbidden`。

![pasted-image-1774289824440.webp](https://i.see.you/2026/03/23/E8td/pasted-image-1774289824440.webp)

猜测是 HTTP Host Header 的问题——Cloudflare 转发过来的请求 Host 不是 `localhost`，被 JupyterLab 拒绝了。好在 ZeroTrust 后台可以设置 Host Header Override。

![pasted-image-1774290133384.webp](https://i.see.you/2026/03/23/3bLz/pasted-image-1774290133384.webp)

改成 `localhost` 之后，终于成功打开了 JupyterLab，拿到了 Terminal，修复了 SSH 配置。

![5.png](https://i.see.you/2026/03/23/eCt5/5.png)

一波三折，总算是把自己从门外救了回来。
