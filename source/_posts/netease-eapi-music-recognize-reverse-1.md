---
title: netease-eapi-music-recognize-reverse-1
date: 2022-06-27 13:46:44
tags:
---
首先是抓了一下协议, 网易云音乐的接口本身是有加密的, 但是没关系, 已经有 [NetEaseCloudMusic](https://github.com/Binaryify/NeteaseCloudMusicApi) 这样的项目逆向出了基本的通讯协议, 可以直接使用.
![surge_mitm.png](https://s2.loli.net/2022/05/05/3BGhneWfKdCQzFm.png)
网易云音乐识曲会向 `https://music.163.com/eapi/music/audio/match?_nmclfl=1` 这个接口发送请求
解密后的请求体大概长这样

```json
{
    "rawdata":"eJx11H9M1HUYB\/C7+2p5\/PhHTUQGwW......",
    "from":"recognize-song",
    "verifyId":1,
    "deviceId":"??????",
    "os":"iOS",
    "header":{

    },
    "algorithmCode":"shazam_v2",
    "times":1,
    "sessionId":"??????",
    "duration":3.4,
    "e_r":true,
    "sceneParams":"{\"action\":0,\"code\":2}"
}
```

![response_decrypted.png](https://s2.loli.net/2022/05/05/23zLWdGwqIacXoy.png)
猜测其中 `rawdata` 就是录音的 Base64 编码, 尝试解码扔进 ffprobe, 但是失败了
有可能 `rawdata` 是根据音频频谱生成了摘要, `algorithmCode` 字段也提示了 `shazam_v2` 这个值
接下来开始逆向 APP
下载了一份比较旧但还能用的 网易云音乐 APK
[old_version_apk.png](https://s2.loli.net/2022/05/05/uliO4SfnUpyIN1L.png)
扔进 Jadx, 发现有比较奇特的混淆方法, 任意地方的字符串都会调用一个函数来进行解密
[Obfuscation.jpg](https://s2.loli.net/2022/05/05/wrAiL2uzyHUsPD7.jpg)
定位到解密函数
[decryptor.png](https://s2.loli.net/2022/05/05/d43Aw5fTby2h9ju.png)
这里的解密看起来并不困难, `C0003a()` 这个类显然是 Base64 解码用的
过我们要写出一个反向的操作(用来方便逆向查找字符串)

```java
import java.util.Base64;

class Main {
    public static String key = "Encrypt";

    public static void main(String[] args) {
        if (args[0].equals("decrypt")) {
            System.out.println(decrypt(args[1]));;
        }else{
            System.out.println(encrypt(args[1]));
        }
    }

    public static String decrypt(String s) {
        byte[] decode = Base64.getDecoder().decode(s);
        String str2;
        int length = decode.length;
        int length2 = key.length();
        int i = 0;
        int i2 = 0;
        while (true) {
            int i3 = i2;
            if (i >= length) {
                break;
            }
            int i4 = i3;
            if (i3 >= length2) {
                i4 = 0;
            }
            decode[i] = (byte) (decode[i] ^ key.charAt(i4));
            i++;
            i2 = i4 + 1;
        }
        str2 = new String(decode);
        return str2;
    }

    public static String encrypt(String str_enc) {
        byte[] s = str_enc.getBytes();
        int length = str_enc.length();
        int length2 = key.length();
        int i = 0;
        int i2 = 0;
        while (true) {
            int i3 = i2;
            if (i >= length) {
                break;
            }
            int i4 = i3;
            if (i3 >= length2) {
                i4 = 0;
            }
            s[i] = (byte) (s[i] ^ key.charAt(i4));
            i++;
            i2 = i4 + 1;
        }
        return Base64.getEncoder().encodeToString(s);
    }
}
```
