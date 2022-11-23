---
title: 主动接收 Google Voice 通知推送
date: 2022-11-23 19:58:35
tags: reverse-engineering
---


## 前言

~~其实只是想研究一下 GCM 通知推送的原理, 顺便看看 GCM 都有些什么有意思的玩法~~

<!-- more -->

## 测试环境

 - Android 12
 - Google Voice 2022.10.31.485627509 (Play Store)
 - macOS Ventura 13.0.1 (22A400)
 - Surge & v2rayNG

## 分析

首先从抓包开始


可以看到首次启动 APP 后直到进入主界面,　Google Voice 客户端会发送多个请求, 其中我们需要关注的请求是向 `https://www.googleapis.com/voice/v1/voiceclient/api2notifications/registerdestination` 这个地址发送的 POST 请求, 其中包含了 GCM 推送使用到的 Token

![register_notification_token.jpg](https://vip2.loli.io/2022/11/23/YxDp2thaAlV4eHC.jpg)

看起来 Google 使用了 Protobuf 来构造请求, 还好这个请求不太复杂,猜想应该也没有签名算法之类的, 我们可以手动构造 Protobuf 包

需要注意的是这个 Protobuf 包还有一个长度参数在第 8 个字节, 从第 10 个字节开始到结束就是 GCM Token 本体

换算一下 `A3` 的十进制表示是 `163`, 正好就是 GCM Token 的长度

![manual_protobuf_generate.jpeg](https://vip2.loli.io/2022/11/23/4girRIuYSAeUoVd.jpg)

成功发送这个注册请求还需要一个 `Header` 中的 `Authorization` 参数, 这个参数是由 `https://android.googleapis.com/auth` 接口生成的, 在 APP 启动和授权 Token 过期时会重新请求生成, 对这个接口的相关讨论可以参考 [StackOverflow](https://stackoverflow.com/questions/22832104/how-can-i-see-hidden-app-data-in-google-drive) 上的讨论

![authorization_header.jpeg](https://vip2.loli.io/2022/11/23/96BEykVL4DIOQq5.jpg)

但是 `https://android.googleapis.com/auth` 接口也会依赖一个 `Token` 参数, 这个参数在设备上登陆 Google 账号时会生成, 所以如果要实现自动化的话, 还要想办法拿到这个参数, 幸好已经 [有人](https://github.com/89z/googleplay) 封装好框架了, 我也在这里封装了 [一个简单的获取 `aas_et` 参数的程序](https://github.com/akinazuki/google_aas_et_generate)

理论上 auth 接口可以给任意使用 [Google 快速登录](https://developers.google.com/identity/one-tap/android/get-saved-credentials) 的 APP 和网页生成请求 Token

现在回到最初的问题上: 我们已经知道了 GCM Token 会以某种形式发送到 Google 的服务器上, Google 服务器在需要的时候再使用这个 Token 来给客户端发送通知,　那有没有办法可以实现一个假的 GCM 客户端来接收 Google 发送的消息？

答案是: 有

众所周知 Chromium 浏览器是开源的, 所以我们可以直接参考 [Chromium 对 GCM 消息的实现](https://chromium.googlesource.com/chromium/chromium/+/trunk/google_apis/gcm) 来实现一个客户端, 太麻烦？ 其实已经有人分析过了 [GCM 的原理](https://medium.com/@MatthieuLemoine/my-journey-to-bring-web-push-support-to-node-and-electron-ce70eea1c0b0), 并且封装了框架 [push-receiver](https://github.com/MatthieuLemoine/push-receiver)

接下来是另一个问题: GCM 生成推送 Token 需要一个 `Sender ID`, 这个 ID 在 GCM 的后台可以看到, 但是我们目前并不知道这个 ID , 当然客户端里肯定有存, 但是 ~~今天不想逆向 APP 代码~~, 而且在我们的问题里有更简单的解决办法: 还是抓包

在 GCM 进行注册的时候会向 `https://android.apis.google.com/c2dm/register3` 发送一个 POST 请求, 这个请求里会包含一个 `sender` 参数, 这个参数就是我们需要的 `Sender ID`

![gcm_sender_id.jpeg](https://vip2.loli.io/2022/11/23/ri1EbK2YzD3cxQy.jpg)

拿到 `sender` 参数之后就可以配合前面的 [push-receiver](https://github.com/MatthieuLemoine/push-receiver) 库来接收 GCM 通知了

使用 push-receiver 库提供的 `register(sender_id)` 函数来注册 GCM Token, 每次使用 `register(sender_id)` 都会产生不同的 GCM Token, 所以生成的 Token 需要持久化保存

注册完成之后就可以使用 `listen(gcm_token, onNotification)` 来注册一个回调事件, 在收到通知后事件就会被调用


## 最终效果

![example_gcm_receive.png](https://vip2.loli.io/2022/11/23/QzN3RGCdKBMJaun.png)

相关代码可参考 [akinazuki/google-voice-notification](https://github.com/akinazuki/google-voice-notification) 与 [akinazuki/google_aas_et_generate](https://github.com/akinazuki/google_aas_et_generate)