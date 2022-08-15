---
title: 网易云音乐听歌识曲 API 逆向 (三)
date: 2022-06-27 20:46:28
tags:
---

闲来无事填一下坑

根据 [这个 issue 的讨论](https://github.com/akinazuki/NeteaseCloudMusic-Audio-Recognize/issues/1)  

现在已经将 [NeteaseCloudMusic-Audio-Recognize](https://github.com/akinazuki/NeteaseCloudMusic-Audio-Recognize) 这个项目移植到了 Node.js上

<!-- more -->

![recognize-and-send-request.png](https://s2.loli.net/2022/06/27/VB1NUkxeAsqn3RD.png)

顺便吐槽一下 NPM 上的 `web-audio-api` 项目  
NPM 版的 `web-audio-api` 已经 7 年没更新了, 直接 `npm install web-audio-api` 安装的包完全不能用  
连 `AudioContext.decodeAudioData()` 方法都没有, 一开始还以为是自己的调用方式有问题  
但是它的示例代码里也是这样写的  
结果一翻 GitHub 上的代码, 发现 GitHub 上安装的版本和 NPM 上的版本完全不一样  
GitHub 上的版本是一直在更新的, 估计有在 Node.js 里解析 `AudioBuffer` 的不少人都会踩这个坑

~~本项目由 Copilot 强力驱动~~