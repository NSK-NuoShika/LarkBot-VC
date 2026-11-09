# LarkBot 使用指南

本文档为面向编写下游程序的开发者的帮助文档。

---

## 目录

- [一、开始之前](#一开始之前)
- [二、快速开始](#二快速开始)
- [三、核心概念](#三核心概念)
- [四、接口参考](#四接口参考)
- [五、派发机制：你的程序要做什么](#五派发机制你的程序要做什么)
- [六、发送文件的详细要点](#六发送文件的详细要点)
- [七、状态码与排障](#七状态码与排障)

---

## 一、开始之前

### 1.1 本程序在做什么

LarkBot 是飞书机器人的一层中间件。它在用户与你的程序之间做消息转发：

1. 接收飞书用户的**单聊文本消息**，按 `程序名 接口名 参数1 参数2...` 接收用户消息，转发给你的程序；     +++++++++++++++++++++++++++++++++++++++++++
2. 接收你的程序发来的**文本或文件**，转发给飞书用户；
3. 维护注册表：维护下游程序的**注册接口**，并定时**清理超时程序**。

<!-- ### 1.2 LarkBot 的部署位置

| 项目 | 说明 |
|---|---|
| 监听地址 | `127.0.0.1:8000`（端口可在 `.env` 的 `PORT` 修改） |
| 访问范围 | 仅本机。你的程序必须与 LarkBot 在同一台机器上运行 |
| 认证 | 无。不要在公网暴露该端口 |
| 响应格式 | 成功 `{"code": 0, "message": "ok", "data": {...}}`；失败 `{"code": <业务码>, "message": "原因", "data": null}`，HTTP 状态码为 4xx/5xx | -->

---

## 二、快速开始

五步接入：

```
① 启动你的 HTTP 服务
        ↓
② 调用 POST /register 注册程序名、接口名、接口路径
        ↓
③ 周期性调用 POST /heartbeat 保持存活
        ↓
④ 提供接口以接收本程序的派发
        ↓
⑤ 按需调用 POST /send/text 或 /send/file，向用户发消息
```

### 2.1 最小可运行示例

下面是一个完整可运行的下游程序示例:

```python
import json
import threading
import time
import urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

LARK_BOT = "http://127.0.0.1:8000"        # LarkBot 示例地址
SELF = "http://127.0.0.1:9100"            # 下游程序示例地址
PROGRAM = "Demo"                          # 程序名
HEARTBEAT_SECONDS = 10                    # 心跳周期


def post(path, payload):
    req = urllib.request.Request(
        LARK_BOT + path,
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read().decode("utf-8"))

# 发送文本
def send_text(target, content):
    return post("/send/text", {"program": PROGRAM, "target": target, "content": content})

# 处理派发
def handle_command(path, sender, args):
    if path == "/echo":
        send_text(sender, "echo: " + " ".join(args))


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        body = json.loads(self.rfile.read(length).decode("utf-8") or "{}")
        sender = body.get("sender", "")
        args = body.get("args", [])

        # 立即回 200
        payload = json.dumps({"code": 0, "message": "ok"}).encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)

        threading.Thread(
            target=handle_command, args=(self.path, sender, args), daemon=True
        ).start()


# 注册一个接口
def register():
    return post("/register", {
        "program": PROGRAM,
        "base_url": SELF,
        "cmd": "echo",
        "path": "/echo",
    })

# 定时发送心跳保活
def heartbeat_loop():
    while True:
        post("/heartbeat", {"program": PROGRAM})
        time.sleep(HEARTBEAT_SECONDS)


def main():
    server = ThreadingHTTPServer(("127.0.0.1", 9100), Handler)
    
    def bootstrap():
        register()
        heartbeat_loop()

    threading.Thread(target=bootstrap, daemon=True).start()
    server.serve_forever()


if __name__ == "__main__":
    main()
```
---

## 三、核心概念

### 3.1 程序名与接口名

| 概念 | 说明 | 唯一性 |
|---|---|---|
| 程序名（`program`） | 你的程序的标识，用户输入命令的第一个词 | 全局唯一，LarkBot 注册的所有下游程序不能重名 |
| 接口名（`cmd`） | 你的程序提供的一个功能 | 同一程序名下唯一，跨程序可重名 |

一个下游程序可以注册多个接口，每个接口调用一次 `/register` 注册，共用同一个程序名和 `base_url`。


### 3.2 参数传递

用户发送格式为 ：'程序名 接口名 参数1 参数2 ···'。

LarkBot 对参数**不做类型转换、校验**，原样以字符串数组传递给你的程序。例如：

| 用户输入 | LarkBot 传给你的 `args` |
|---|---|
| `Demo echo Hello` | `["Hello"]` |
| `Demo echo 你好 世界` | `["你好", "世界"]` |
| `Demo echo a   b`（多个空格） | `["a", "b"]` |
| `Demo echo`（无参数） | `[]` |

### 3.3 用户 id

LarkBot 派发时会把发送者身份放在 `sender` 字段里。它是飞书的 **open_id**，即用户的唯一标识。

你按需把 `sender` 原样存下来，之后调用 `/send/text` 或 `/send/file` 发送时传回去，消息会发给对应的用户。

### 3.4 异步

`/send/text` 与 `/send/file` 在通过校验后立即返回响应体，此时消息只是进了 LarkBot 的发送队列，不代表成功发送。

---

## 四、接口参考

<!-- 所有接口都返回 HTTP 200 + `code: 0` 表示成功，其余情况 HTTP 4xx/5xx + 非 0 业务码。完整业务码见[第八章](#八状态码与排障)。 -->

### 4.1 `POST /register` —— 注册程序与接口

每次调用注册一个接口。程序有多个接口时，按接口数量多次调用。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `program` | string | 是 | 程序名，全局唯一。 |
| `base_url` | string | 是 | 程序地址，以 `http://` 或 `https://` 开头 |
| `cmd` | string | 是 | 接口名，同一程序内唯一。 |
| `path` | string | 是 | 接口路径，以 `/` 开头，如 `/query` |

- 请求示例
  ```json
   POST /register
  {"program": "基金", "base_url": "http://127.0.0.1:9001", "cmd": "查询", "path": "/query"}
  ```

- 响应

  首次注册成功：

  ```json
  200 {"code": 0, "message": "ok", "data": {"program_id": 100000}}
  ```

  同程序且同接口再次注册：

  ```json
  200 {"code": 1001, "message": "注册成功（覆盖）", "data": {"program_id": 100000}}
    ```

- 行为要点
  - 注册同时会刷新该程序的存活时间。



### 4.2 `POST /heartbeat` —— 心跳

周期性调用，用于维持程序的存活。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `program` | string | 是 | 已注册的程序名 |

- 请求示例

    ```json
    POST /heartbeat
    {"program": "基金"}
    ```

- 响应

    ```json
    200 {"code": 0, "message": "ok", "data": null}
    404 {"code": 0, "message": "程序未注册: 基金", "data": null}
    ```

- 行为要点

  - 心跳只刷新存活时间，不修改 `base_url` 或接口表。
  - 返回 `404` 说明程序已被超时清除或从未注册，必须重新执行 `/register`。

### 4.3 `POST /send/text` —— 发送文本

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `program` | string | 是 | 程序名，必须已注册 |
| `target` | string | 是 | 接收用户的 open_id |
| `content` | string | 是 | 文本内容 |

- 请求示例

    ```json
    POST /send/text
    {"program": "基金", "target": "ou_55e1a0c9...", "content": "今日收益 +2.3%"}
    ```

- 响应

    ```json
    200 {"code": 0, "message": "ok", "data": {"queued": true}}
    400 {"code": 4001, "message": "程序未注册: 基金", "data": null}
    ```

- 行为要点

  - `program` 必须是已注册的程序名，否则返回 `4001`。。
  - `target` 必须是有效的 open_id，LarkBot 不做格式预校验。open_id 写错时接口仍返回成功，但飞书侧发送失败，消息丢失。

### 4.4 `POST /send/file` —— 发送文件

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `program` | string | 是 | 程序名，必须已注册 |
| `target` | string | 是 | 接收用户的 open_id |
| `file_name` | string | 是 | 文件名，带扩展名 |
| `content_base64` | string | 是 | 文件内容的标准 base64 编码（不是 URL-safe 变体） |
| `duration_ms` | int | 仅 mp4 必填 | 视频时长（毫秒）。非 mp4 忽略 |

请求示例
    普通示例：

    ```json
    POST /send/file
    {"program": "基金", "target": "ou_55e1a0c9...", "file_name": "报告.xlsx",
    "content_base64": "UEsDBBQABgAIAAAAIQ..."}
    ```

    mp4 示例：

    ```json
    POST /send/file
    {"program": "基金", "target": "ou_55e1a0c9...", "file_name": "行情.mp4",
    "content_base64": "AAAAGGZ0eXBpc29t...", "duration_ms": 12500}
    ```

- 响应

    ```json
    200 {"code": 0, "message": "ok", "data": {"queued": true}}
    400 {"code": 4003, "message": "文件名不合法（含缺扩展名）", "data": null}
    ```

---

## 五、派发机制：你的程序要做什么

### 5.1 暴露的端点

对每个注册过的接口，你的程序要在 `base_url + path` 上暴露一个 POST 端点，接收 JSON：

| 字段 | 类型 | 说明 |
|---|---|---|
| `sender` | string | 发送者的 open_id |
| `args` | string[] | 用户命令里接口名之后的参数，按空白切分；无参数时为 `[]` |

请求示例：

```json
POST http://127.0.0.1:9001/query
{"sender": "ou_55e1a0c9...", "args": ["沪深300"]}
```

### 5.2 响应约定

LarkBot 需要两个字段：

1. HTTP 状态码；
2. 响应体里的 `code` 字段（取不到就记空）。

LarkBot 不解析响应体里的其他内容，也不会把响应体转给用户。

推荐的响应：

```json
200 {"code": 0, "message": "ok"}
```

### 5.3 超时

LarkBot 调用你的端点有硬性时限，取决于配置。超时后，放弃派发，不重试：

- LarkBot 在 `dispatch.log` 记一条「发送失败」；
- 本次用户命令被丢弃，不会重试。

因此你的端点需要尽快返回。如果业务处理耗时较长，推荐模式是：成功收到派发 → 立即返回 200 → 后台线程处理

### 5.4 失败与重试

LarkBot 不做任何重试。无论是派发还是发送飞书，任何发送失败（例如未查询到下游路由、网络抖动、超时等），都会被忽略且不重试。

---

## 六、发送文件的详细要点

### 6.1 base64 编码

- 必须使用标准 base64（字符集 `A-Z a-z 0-9 + /`，可带 `=` 填充），不能使用 URL-safe 变体（`-`、`_`）。
- 编码后不要插入换行。带换行的 MIME 风格 base64 会被判为非法（4004）。
- 发送前请先确认解码后不为空，空内容返回 4004。

### 6.2 大小限制

- 上限 30MB（`30 × 1024 × 1024 = 31,457,280` 字节），指解码后的文件大小。

### 6.3 mp4 视频

- 必须传 `duration_ms`，单位毫秒，且必须大于 0。
- 封面由 LarkBot 自动生成并附带，不需要提供。

### 6.4 文件名

- 只传文件名，不要带路径。
- 文件名里避免出现换行、制表符等控制字符。

---

## 七、状态码与排障

### 7.1 业务码表

| 业务码 | 分段 | 含义 |
|---|---|---|---|
| 0 | 通用 | 成功，或无需业务码 |
| 1001 | 成功提示 | 同程序同接口重复注册，发送覆盖| 
| 4001 | 客户端 | 程序未注册（含已被心跳清除） | 
| 4003 | 客户端 | 文件名缺少扩展名 |
| 4004 | 客户端 | base64 非法或解码后为空 |
| 4005 | 客户端 | 文件超过 30MB |
| 4006 | 客户端 | mp4 缺少 `duration_ms` |
| 4007 | 客户端 | 请求不合法（字段缺失、类型错误、取值非法） |
| 5000 | 服务端 | LarkBot 内部错误 |

---
