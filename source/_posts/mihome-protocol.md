---
title: 米家 Android 客户端通讯协议
date: 2023-07-01 00:00:00
tags: reverse-engineering
---


## 前言

大约四年前，我与[朵姐姐](https://keep.moe/)一起通过逆向工程揭示了[米家客户端旧版的通讯协议](https://github.com/akinazuki/mijia-api/tree/old)的秘密。然而，随着米家客户端的更新，这一旧版的通讯协议已经不再适用。

本文基于米家 Android 8.5.704 客户端 (Google Play)

<!-- more -->

## 返回的解密实现

![current-encryption.png](https://vip2.loli.io/2023/07/01/Nx3YnCUI5AiOsHS.png)

新版的加密算法增加了一个 `rc4_hash__` 字段，同时 `data` 字段从明文 JSON 转变为了一段密文。

在详细了解之前，我们首先使用 jadx 工具进行代码审查。

![jadx-search-rc4-hash.png](https://vip2.loli.io/2023/07/01/G6bKlISVZCiYyMT.png)

全局搜索字符串 `rc4_hash__` 找到了这几个调用的地方

![sha1-crypto.png](https://vip2.loli.io/2023/07/01/HYlxq2VuyW5Emtj.png)

这看起来是一个用于为特定参数生成 SHA-1 签名的方法。接着我们尝试使用 Frida Hook 来验证我们的猜测。

![frida-hook.png](https://vip2.loli.io/2023/07/01/ACIwvrDoRfpOyz5.png)

经过分析，我们得出这个方法的签名大概如下：

```
sign(method: string, path: string, params: { [key: string]: string }, key: string) : string
```

看起来这个 Key 是动态生成的, 继续往前追踪

![key-trace.png](https://vip2.loli.io/2023/07/01/br2sXgy9k5hCVEB.png)

这个 key 由第 163 行的函数生成，而这个函数又依赖于第 158 行生成的 `nonce` 参数，函数生成的结果是一个经过 SHA-256 摘要的字符串，通过对合并 Buffer 的函数进行 Hook, 可以得知第一个参数 `miServiceTokenInfo2.OooOo0O` 是一个始终固定的值，在客户端中的定义叫做 `ssecurity` ，登陆米家时会从服务端获得并保存

```java
String nonce = i0a.OooO00o(miServiceTokenInfo2.OooOo0o); // 根据当前的时间戳除以 60000 并返回字符串
String key = Version.Oooo0( // 转换为 Base-64 String
  Version.o000OO0O( // SHA-256 Buffer
    OooO0OO( //合并两个 Buffer
      h0a.OooO00o(miServiceTokenInfo2.OooOo0O), // ssecurity Token
      h0a.OooO00o(nonce)
    )
  )
);
```

这个 key 在两个地方被使用：一个是上面提到的 `sign` 方法，另一个是第 170 行的 `k0a k0aVar = new k0a(key);` 这个类会使用 `key` 来构造一个加密解密的实例，我们将其跟踪进一步查看。

![rc4-wrapper.png](https://vip2.loli.io/2023/07/01/f1L3NGwc47quxVY.png)

猜测被证实，它初始化 `RC4Encryption` 实例后使用 1024 byte 的 0x00 来调用一次 `encrypt()` 函数。

但是解密函数有一些独特，调用解密函数后，一个 bool 值将决定是否执行特定的操作。

![caller.png](https://vip2.loli.io/2023/07/01/sm8vopMbAHnEK4w.png)

看起来是 GZIP 有关系, 返回的密文最外层是 RC4, 解密了之后就是一段 GZIP 流, GZIP 内容解压之后的内容应该就是明文的数据了


![manual-decrypt.png](https://vip2.loli.io/2023/07/01/9NxdMHl5wFbrfo4.png)

尝试手动使用同样的参数解密一下试试看

![decryption-code.png](https://vip2.loli.io/2023/07/01/oMyqtpWa6cIkXv5.png)

那么返回的解密实现就完成了

## 请求的加密实现

回顾前述的签名函数，我们可以发现它会被调用两次。首先，签名函数被用来计算 JSON body，接着，计算完整请求中的 signature 字段。以下是一个具体的例子：


```
Invoke Signature:  POST /v2/homeroom/gethome with key Key: rf9CDGE+jEWDayyJ9CXzm5HlBwpGpI1rmFXiQdLftUU=
key: data, value: {"fg":false,"fetch_share":true,"fetch_share_dev":true,"limit":300,"app_ver":7,"fetch_cariot":true}
Invoke Signature result: NXh9A8ImXOBXCGiJ9KiPGtUsjkg=


Invoke Signature:  POST /v2/homeroom/gethome with key Key: rf9CDGE+jEWDayyJ9CXzm5HlBwpGpI1rmFXiQdLftUU=
key: data, value: WGhH5jrYyj4aeqJ9nYCHJumbC23iVhBhOlvIns690YuWDTgeSASv+wh2uPFeaaElpdXfQz63s4Iv3MkTskmiIzafUxQNurIf+I8Y2Y3XfJuC93yBjFld9lA3cP3jVIbhmjY=
key: rc4_hash__, value: Ng9eDjonPejap1L7cM/Zo/56jgF6z7KvkEgRWA==
Invoke Signature result: N9ZuY3flWrPWb6RnUaTU77bJ4Ak=

```

在第一次调用过程中，只有 `data` 字段的明文。然后，第二次调用时，使用先前得到的 `rc4_hash__` 以及 `data` 字段，将这些数据经过 RC4 流加密，然后再作为签名参数计算，最终得到 `signature`。下面是部分代码实现：

```typescript
async buildSign(path: string, data: any) {
  const nonce = this.generateNonce()
  const key = await this.getRC4Key(nonce)
  const query = {
    data: JSON.stringify(data),
  }
  query["rc4_hash__"] = await this.sign('POST', path, query, key.toString('base64'))
  const streamendEncrypt = await this.streamendEncrypt(nonce)
  for (const key of Object.keys(query)) {
    const value = query[key]
    const encrypted = streamendEncrypt.encryptDecrypt(Buffer.from(value))
    query[key] = encrypted.toString('base64')
  }
  query["signature"] = await this.sign('POST', path, query, key.toString('base64'))
  query["_nonce"] = nonce.toString('base64')
  return {
    query,
    nonce,
    key,
  }
}
```

至此，我们完成了请求的加密实现。要构造请求所需要的 `ssecurity` 参数，我们可以通过抓取 Android 客户端下 `api.io.mi.com` 子域名的请求来获取（这看起来是一个 bug，因为在 Android 端的代码实现中，`ssecurity` 也作为参数提交到 API，但这个参数实际上并不需要被用到。在 iOS 的实现中，请求并没有带上这个参数）。如果无法通过抓包获取 `ssecurity` 参数，也可以通过模拟登录等方式获取，但这里不再详述。

关于加密和解密的具体实现，可以参考 [akinazuki/mijia-api/proxy.ts](https://github.com/akinazuki/mijia-api/blob/master/proxy.ts)


## 后记

上一次逆向 Android 程序还是用的 Xposed, 每次都要重新编译一个 Xposed 模块, 然后再安装到手机上, 这次使用 Frida 之后, 发现 Frida 的使用体验要好很多, 也不需要重新编译模块, 只需要在电脑上安装好 Frida, 然后在手机上安装好 Frida Server, 就可以直接使用了

## 参考

[早苗の魔导典 - frida的配置与使用(从0到hook淘宝签名算法）](https://moe.me/2021/08/16/frida%E7%9A%84%E4%BD%BF%E7%94%A8%E4%B8%8E%E9%85%8D%E7%BD%AE/)