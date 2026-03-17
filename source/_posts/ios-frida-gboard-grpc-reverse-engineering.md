---
title: iOS Frida 初尝试之分析 Gboard 搜索接口
date: 2026-03-17 00:00:00
tags: reverse-engineering
---

最近在折腾 Gboard（Google 键盘）的逆向，想看看它在用户搜索的时候到底和 Google 的服务器交换了些什么数据。一开始的想法是挂代理抓包，但 Surge 和 Reqable 都没能解出来——Gboard 走的是 HTTP/2 + gRPC，MITM 代理基本没法正常解析。最终通过 Frida hook BoringSSL 解密越狱 iPhone 上的 TLS 流量，逐层拆解 HTTP/2、gRPC 和 Protobuf，还原出了完整的搜索接口并用 grpcurl 复现了请求。

<!-- more -->

## 先抓个包看看

虽然代理解不了内容，但至少可以先确认 Gboard 在和谁通信。macOS 下可以用 `rvictl` 给 USB 连接的 iOS 设备创建一个虚拟网络接口，`tcpdump` 抓一下：

```bash
rvictl -s <UDID>
# Starting device <UDID> [SUCCEEDED] with interface rvi0

sudo tcpdump -i rvi0 -w capture.pcap tcp
```

在手机上用 Gboard 搜了几下，停掉抓包，丢进 Wireshark 一看

![pasted-image-1773752621753.webp](https://i.see.you/2026/03/17/7Rdr/pasted-image-1773752621753.webp)

通信走的 TLS 1.3，端口 443，HTTP/2。到这一步确认了目标域名，但具体通信内容还是密文。

## Frida 登场

既然手机已经越狱了，那就不客气了。iOS 底层用的是 BoringSSL，可以用 Frida 来获取 TLS 明文。这里有两种思路：

- **导出 TLS session keys**：让 Wireshark/tshark 直接解密 pcap
- **Hook SSL_read/SSL_write**：在 Frida 端解析 HTTP/2 帧后回传

先说第一种，更适合调试。

不过在开始之前，有一个坑需要注意：**Gboard 的键盘扩展和主 app 是两个不同的进程**，网络请求是在扩展里发出的，attach 到主进程上什么都抓不到：

```bash
$ frida-ps -U | grep Gboard
2688  Gboard    # 主 app
2653  Gboard    # 键盘扩展 ← 要 attach 这个
```

### 方法一：导出 TLS Session Keys

通过 Frida hook `SSL_CTX_set_keylog_callback`，给每个 SSL_CTX 注入一个回调，让 BoringSSL 在握手时把密钥材料吐出来：

```javascript
var SSL_CTX_set_keylog_callback = Module.findExportByName(null, "SSL_CTX_set_keylog_callback");
var SSL_get_SSL_CTX = new NativeFunction(
    Module.findExportByName(null, "SSL_get_SSL_CTX"), 'pointer', ['pointer']);
var setKeylogCallback = new NativeFunction(
    SSL_CTX_set_keylog_callback, 'void', ['pointer', 'pointer']);

var keylogCallback = new NativeCallback(function(ssl, line) {
    send({ type: "keylog", line: line.readUtf8String() });
}, 'void', ['pointer', 'pointer']);

var hookedCtx = new Set();

Interceptor.attach(Module.findExportByName(null, "SSL_new"), {
    onLeave: function(retval) {
        var ctx = SSL_get_SSL_CTX(retval);
        if (!hookedCtx.has(ctx.toString())) {
            hookedCtx.add(ctx.toString());
            setKeylogCallback(ctx, keylogCallback);
        }
    }
});
```

Python 端接收后写入 NSS Key Log 格式的文件：

```
CLIENT_HANDSHAKE_TRAFFIC_SECRET b169d8... 40410c...
SERVER_HANDSHAKE_TRAFFIC_SECRET b169d8... 81442a...
CLIENT_TRAFFIC_SECRET_0 b169d8... 2cc598...
SERVER_TRAFFIC_SECRET_0 b169d8... fbce94...
EXPORTER_SECRET b169d8... fdbf0f...
```

注意需要用 spawn 模式重启 Gboard，确保 keylog callback 在 TLS 握手之前就已经注入，否则已建立的连接不会触发回调。

配合 `rvictl` + `tcpdump` 同时抓包，然后用 tshark 解密：

```bash
tshark -r capture.pcap \
  -o "tls.keylog_file:keylog.txt" \
  -z "follow,tls,hex,0" -q
```

输出中可以直接看到解密后的 HTTP/2 明文：

```
00000052  00 00 00 04 01 00 00 00  00 00 01 45 01 04 00 00  ........  ...E....
00000062  00 01 40 05 3a 70 61 74  68 3a 2f 67 6f 6f 67 6c  ..@.:pat  h:/googl
00000072  65 2e 69 6e 74 65 72 6e  61 6c 2e 6d 6f 74 68 65  e.intern  al.mothe
00000082  72 73 68 69 70 2e 67 62  6f 61 72 64 2e 76 31 2e  rship.gb  oard.v1.
00000092  47 62 6f 61 72 64 53 65  72 76 69 63 65 2f 53 65  GboardSe  rvice/Se
```

不过有一点需要注意：Gboard 的 ALPN 协商结果是 `grpc-exp` 而不是 `h2`，tshark 不会自动识别为 HTTP/2 协议。`follow,tls` 可以看到解密后的原始数据流，但如果想在 Wireshark GUI 中看到结构化的 HTTP/2 帧解析，需要手动 `Decode As → HTTP2`。

### 方法二：Hook SSL_read/SSL_write

如果需要实时解析和过滤，可以直接在 Frida 端 hook SSL 函数、拆解 HTTP/2 帧后回传。这种方式不需要同时跑 tcpdump，所有解析都在脚本里完成。

```python
device = frida.get_usb_device()
session = device.attach(2653)  # 键盘扩展的 PID
script = session.create_script(AGENT_JS)
script.on('message', on_message)
script.load()
```

Frida 注入的 JS 脚本负责 hook SSL 函数、过滤目标流量、解析 HTTP/2 帧并传回 Mac：

```javascript
var tracked = new Set();
var sslBuffers = {};

function getKey(ssl, dir) { return ssl.toString() + "_" + dir; }

["SSL_write", "SSL_read"].forEach(function(fname) {
    var addr = Module.findExportByName(null, fname);
    if (!addr) return;
    Interceptor.attach(addr, {
        onEnter: function(args) {
            this.ssl = args[0];
            this.buf = args[1];
            this.len = args[2].toInt32();
            this.fname = fname;
        },
        onLeave: function(retval) {
            var n = retval.toInt32();
            if (n <= 0) return;
            var data = this.buf.readByteArray(n);
            var arr = new Uint8Array(data);
            var dir = this.fname === "SSL_write" ? "req" : "resp";

            var str = "";
            try { str = this.buf.readUtf8String(n > 4096 ? 4096 : n); } catch(e) {}
            if (str && (str.indexOf("gboard-pa") !== -1
                     || str.indexOf("x-goog-api-key") !== -1)) {
                tracked.add(this.ssl.toString());
            }
            if (!tracked.has(this.ssl.toString()) && tracked.size > 0) return;

            var key = getKey(this.ssl, dir);
            if (!sslBuffers[key]) sslBuffers[key] = [];
            for (var i = 0; i < arr.length; i++) sslBuffers[key].push(arr[i]);

            // 跳过 HTTP/2 connection preface
            var buf = sslBuffers[key];
            if (buf.length >= 24) {
                var preface = "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n";
                var match = true;
                for (var p = 0; p < preface.length && p < buf.length; p++) {
                    if (buf[p] !== preface.charCodeAt(p)) { match = false; break; }
                }
                if (match) buf.splice(0, 24);
            }

            // 按 HTTP/2 帧边界切割并发送
            while (buf.length >= 9) {
                var frameLen = (buf[0] << 16) | (buf[1] << 8) | buf[2];
                var totalLen = 9 + frameLen;
                if (frameLen > 65536) { buf.splice(0, 1); continue; }
                if (buf.length < totalLen) break;

                var ftype = buf[3];
                var flags = buf[4];
                var sid = ((buf[5] & 0x7f) << 24) | (buf[6] << 16)
                        | (buf[7] << 8) | buf[8];
                var payload = buf.slice(9, totalLen);
                buf.splice(0, totalLen);

                if (ftype === 0) { // DATA
                    send({ t: "d", dir: dir, sid: sid, flags: flags },
                         new Uint8Array(payload).buffer);
                } else if (ftype === 1) { // HEADERS
                    send({ t: "h", dir: dir, sid: sid, flags: flags },
                         new Uint8Array(payload).buffer);
                }
            }
            sslBuffers[key] = buf;
        }
    });
});
```

Mac 端的 Python 脚本负责接收帧、按 stream 累积 DATA、在收到 trailers 时解码 gRPC：

```python
import frida, struct, gzip

stream_bufs = {}  # {stream_id}_{dir} -> bytearray

def flush_stream(sid, direction):
    """收到 END_STREAM 后，拆 gRPC 帧"""
    key = f'{sid}_{direction}'
    if key not in stream_bufs or len(stream_bufs[key]) == 0:
        return
    buf = bytes(stream_bufs[key])
    stream_bufs[key] = bytearray()

    # gRPC 帧: 1B 压缩标志 + 4B 长度 + payload
    compressed = buf[0]
    msg_len = struct.unpack('>I', buf[1:5])[0]
    payload = buf[5:5 + msg_len]

    if compressed == 1:
        payload = gzip.decompress(payload)

    # payload 就是 protobuf 明文，可以保存或解码
    with open(f's{sid}_{direction}.bin', 'wb') as f:
        f.write(payload)

def on_message(message, data):
    if message['type'] != 'send' or data is None:
        return
    p = message['payload']
    sid, d, flags = p['sid'], p['dir'], p['flags']

    if p['t'] == 'h':  # HEADERS 帧
        # gRPC 用 trailers（带 END_STREAM 的 HEADERS）标记响应结束
        if flags & 0x1:
            flush_stream(sid, d)

    elif p['t'] == 'd':  # DATA 帧
        key = f'{sid}_{d}'
        if key not in stream_bufs:
            stream_bufs[key] = bytearray()
        stream_bufs[key].extend(data)
        # 请求的 DATA 帧通常直接带 END_STREAM
        if flags & 0x1:
            flush_stream(sid, d)

device = frida.get_usb_device()
session = device.attach(2653)
script = session.create_script(AGENT_JS)
script.on('message', on_message)
script.load()
```

这里的关键是 `flush_stream` 的触发时机

```
在 gRPC over HTTP/2 中：
	请求流在最后一个帧上携带 END_STREAM，该帧可以是 DATA（有请求体时）或 HEADERS（无请求体时）。
	响应流必须以 trailers（HEADERS 帧）结束，并在该帧上携带 END_STREAM，且其中包含 grpc-status。
```

## 拆协议

Gboard 的搜索请求套了好几层协议：

```
TLS → HTTP/2 帧 → gRPC 帧 → Protocol Buffers
```

### HTTP/2

以实际捕获到的一个 HEADERS 帧为例：

| 字段 | 字节 | 值 | 说明 |
|------|------|-----|------|
| Length | `00 01 45` | 325 | Payload 长度 |
| Type | `01` | HEADERS | 帧类型 |
| Flags | `04` | END_HEADERS | 标志位 |
| Stream ID | `00 00 00 01` | 1 | 流 ID |
| Payload | `...` | (325 bytes) | HPACK 编码的请求头 |

解析这个 HEADERS 帧的 payload 之后，关键信息就出来了：

```
:path: /google.internal.mothership.gboard.v1.GboardService/Search
:authority: gboard-pa.googleapis.com:443
content-type: application/grpc
x-goog-api-key: AIzaSyAW1OaFX_qzPLnqhRUFdPLHxRen3dxEwLI
```

### gRPC

HTTP/2 DATA 帧的 payload 里面是 gRPC 的封装。以捕获到的一个请求 DATA 帧为例：

| 字段 | 字节 | 值 | 说明 |
|------|------|-----|------|
| Compressed | `00` | 0 | 未压缩 |
| Length | `00 00 01 b8` | 440 | Protobuf 消息长度 |
| Message | `0a 12 e6 88 91...` | (440 bytes) | Protobuf 编码的请求体 |

响应的 Compressed 标志为 `01`（gzip 压缩），需要先解压才能拿到 protobuf 数据。

注意：gRPC 的 `END_STREAM` 标志不在 DATA 帧上，而在后面的 **trailers HEADERS 帧**上。

### Protobuf

最后一层。因为没有 `.proto` 定义文件，只能对着 wire format 手动解析。Protobuf 的每个字段以一个 varint tag 开头，低 3 位是 wire type，高位是 field number：

```python
tag, offset = read_varint(data, 0)
field_number = tag >> 3  # 字段编号
wire_type = tag & 7      # 0=varint, 2=length-delimited, ...
```

wire type 2 比较烦人，string、bytes 和嵌套 message 用的都是同一个类型，只能靠尝试 UTF-8 解码和递归解析来猜。

不过好在 Gboard 的数据结构并不复杂，几轮解码下来，请求和响应的结构就清楚了。

## 接口结构

### 请求

搜索 "东京塔" 时发出的请求长这样：

```protobuf
message SearchRequest {
  string query = 1;             // "东京塔"
  ClientInfo client_info = 2;
  int32 search_type = 3;        // 1
}

message ClientInfo {
  string version = 2;           // "2.3.19.444584732"
  AuthToken auth_token = 3;     // 会话 token
  string user_agent = 4;        // 伪装的浏览器 UA
  Locale locale = 5;            // { language: "zh_cn", country: "JP", domain: ".google.com" }
}
```

### 响应

响应是一个搜索结果的列表，每条包含标题、域名、摘要和操作按钮：

```protobuf
message SearchResponse {
  repeated SearchResult results = 1;
}

message SearchResult {
  ResultInfo info = 1;           // { title, domain, snippet }
  repeated Action actions = 8;   // { action_type, url, label }
  AnswerCard answer_card = 19;   // 精选答案卡片（可选）
}
```

比较有意思的是 `AnswerCard` 这个字段。如果搜的是"日落时间是几点"之类的问题，响应里会直接带一个答案卡片：

```
value: "17:51"
context: "2026年3月17日星期二 GMT+9"
location: "东京都町田市日落"
```

而 `Action` 则有两种类型：`action_type=1` 是分享（生成 `gboard://share/` 链接），`action_type=7` 是原始 URL 链接。

## 复现

接口结构搞清楚了，接下来就是验证。

```protobuf
syntax = "proto3";

package google.internal.mothership.gboard.v1;

service GboardService {
  rpc Search (SearchRequest) returns (SearchResponse);
}

message SearchRequest {
  string query = 1;
  ClientInfo client_info = 2;
  int32 search_type = 3;
}

message ClientInfo {
  string version = 2;
  AuthToken auth_token = 3;
  string user_agent = 4;
  Locale locale = 5;
}

message AuthToken {
  string token = 2;
}

message Locale {
  string language = 1;
  string country = 2;
  string google_domain = 3;
}

message SearchResponse {
  repeated SearchResult results = 1;
  ResponseMetadata metadata = 2;
}

message SearchResult {
  ResultInfo info = 1;
  repeated Action actions = 8;
  AnswerCard answer_card = 19;
}

message ResultInfo {
  string title = 1;
  string domain = 2;
  string snippet = 3;
  RatingInfo rating = 4;
  int32 has_rating = 6;
}

message RatingInfo {
  float score = 1;
  float max_score = 2;
  float unknown_float = 3;
  int32 review_count = 5;
  string review_text = 7;
}

message Action {
  int32 action_type = 1;  // 1=分享, 7=外部打开
  string url = 2;
  string label = 3;
}

message AnswerCard {
  int32 type = 1;
  string value = 2;
  string context = 3;
  string location = 4;
}

message ResponseMetadata {
  bytes unknown = 2;
}
```

有了 proto 文件，用 `grpcurl` 可以直接在命令行验证：

```bash
grpcurl \
  -proto gboard_service.proto \
  -H 'x-goog-api-key: AIzaSyAW1OaFX_qzPLnqhRUFdPLHxRen3dxEwLI' \
  -d '{
    "query": "凋叶棕",
    "client_info": {
      "version": "2.3.19.444584732",
      "auth_token": {"token": "529=hAKYnp4v......."},
      "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 16_5 like Mac OS X) AppleWebKit/600.1.1 (KHTML, like Gecko) GBoard GKB/2.3.19.444584732 Mobile/3B143 Safari/600.1.1",
      "locale": {"language": "zh_cn", "country": "JP", "google_domain": ".google.com"}
    },
    "search_type": 1
  }' \
  gboard-pa.googleapis.com:443 \
  google.internal.mothership.gboard.v1.GboardService/Search

{
  "results": [
    {
      "info": {
        "title": "凋叶棕",
        "domain": "www.rd-sounds.com",
        "snippet": "音楽同人サークル「RD-Sounds」及び「凋叶棕」のサイトです。"
      },
      "actions": [
        {
          "actionType": 1,
          "url": "gboard://share/?text=https://www.rd-sounds.com/%0A%E5%87%8B%E5%8F%B6%E6%A3%95%0A",
          "label": "分享"
        },
        {
          "actionType": 7,
          "url": "https://www.rd-sounds.com/",
          "label": "使用外部应用查看"
        }
      ]
    },
    .......
  ],
  "metadata": {}
}
```
