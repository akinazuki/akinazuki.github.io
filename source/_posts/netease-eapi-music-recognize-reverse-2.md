---
title: 网易云音乐听歌识曲 API 逆向 (二)
date: 2022-05-12 00:33:26
tags:
---


前两天看到网易云音乐发布了一个网页上做音乐识别的 [Chrome 插件](https://juejin.cn/post/7094083239702659109)

![netease-chrome-recognize](https://s2.loli.net/2022/05/12/sdvE3LHGx19owOY.png)

<!-- more -->

于是立即下载了一份来研究

![source](https://s2.loli.net/2022/05/12/XwcgmIPYsuLpCBf.png)

Chrome 插件的请求方式和结构和客户端几乎一模一样

![request-struct](https://s2.loli.net/2022/05/12/feCIWNdhlBVJXML.jpg)

根据函数栈追踪, 可以分析出比较核心的逻辑都在 `sandbox.bundle.js` 内

看起来是用了 WebAssemble 来解析网页录音, 在点击开始录音后会开始录制当前 TAB 的音频

当录音完成后就会将音频传到 WASM 暴露的函数 `l().ExtractQueryFP(...)` 中

![parse-buffer](https://s2.loli.net/2022/05/12/Ljm8hg5Kztv9VHW.jpg)

WASM 层处理完成后就会将返回的 ArrayBuffer 封装成一个 Base64 串, 最后会将这个 Base64 串提交给 API

![request-code](https://s2.loli.net/2022/05/12/TEWq917khpOKZPd.jpg)

* * *

虽然还是不太清楚 WASM 层内部是如何处理传入的音频数据, 但是已经可以将它的代码抽出来作为一个类库了

基于插件代码制作了一份 DEMO, 可以参考 [NeteaseCloudMusic-Audio-Recognize](https://github.com/akinazuki/NeteaseCloudMusic-Audio-Recognize)

`rec.json` 内是封装成 JSON 的录音 ArrayBuffer

运行后会在 Chrome Console 打印出音频指纹的 Base64 串

[本页面已更新](/netease-eapi-music-recognize-reverse-3/)

![console](https://s2.loli.net/2022/05/12/ZuSIAYHMtlxgpsk.png)

![postman-test](https://s2.loli.net/2022/05/12/UmwNMRthrfbzQiJ.jpg)