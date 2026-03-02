# 验证：Context Lens 是否真正把请求转发到自定义 OpenAI-Compatible 上游（curl 版）

> 目的：不依赖 opencode，直接验证 `UPSTREAM_OPENAI_URL` 是否在 **Context Lens proxy (4040)** 生效，并且在落盘/界面里能看到正确的上游 URL 与请求信息。
>
> 适用场景：
>
> - 你使用自定义模型与自定义 URL（OpenAI-compatible 网关）。
> - 发现 Context Lens 记录的 `http.url/targetUrl/model` 与预期不一致。
> - opencode 场景存在 MITM / SSE（`text/event-stream`）导致 raw body 不落盘，难以确认请求细节。

---

## 0. 前置条件

- 你已能访问 Context Lens proxy：`http://127.0.0.1:4040`
- 你希望把上游指到：

```bash
http://model.mify.ai.srv/v1
```

- 你机器上有：`curl`、`jq`（用于检查落盘 `.lhar`）

---

## 1. 启动 Context Lens（proxy + UI）

在任意终端执行：

```bash
context-lens background start
context-lens background status
```

然后打开 UI：

- http://localhost:4041

> 若你不想后台模式，也可以用你自己的启动方式，只要确保 4040/4041 正常即可。

---

## 2. 设置自定义上游（只对当前终端生效）

在**将要执行 curl 的终端**里：

```bash
export UPSTREAM_OPENAI_URL="http://model.mify.ai.srv/v1"
```

> 说明：`UPSTREAM_OPENAI_URL` 必须在 Context Lens **proxy 进程启动时**就存在才会生效。
> 
> 如果你是先 `background start`，再 `export`，那么后台 proxy 很可能没拿到这个变量。
> 
> ✅ 最稳妥做法：先 stop，再带环境变量 start：
>
> ```bash
> context-lens background stop
> UPSTREAM_OPENAI_URL="http://model.mify.ai.srv/v1" context-lens background start
> ```

---

## 3. 发送一个非流式 Chat Completions 请求（最通用）

```bash
curl -i -N \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TEST" \
  http://127.0.0.1:4040/v1/chat/completions \
  -d '{
    "model": "xiaomi/mimo-v2-flash",
    "messages": [
      { "role": "user", "content": "ping" }
    ],
    "stream": false
  }'
```

### 你应该看到什么

- 如果上游能处理该请求，你会收到 `HTTP/1.1 200 ...`（或上游业务错误，但至少说明转发到你指定的上游了）。
- 去 UI（http://localhost:4041）查看最新 session/entry：
  - `http.url/targetUrl` 应该包含 `model.mify.ai.srv`（而不是 `api.openai.com` / `opencode.ai`）。

---

## 4. 如果你上游只支持 Responses API：发送 /v1/responses

```bash
curl -i -N \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TEST" \
  http://127.0.0.1:4040/v1/responses \
  -d '{
    "model": "xiaomi/mimo-v2-flash",
    "input": "ping",
    "stream": false
  }'
```

同样去 UI 看 `http.url/targetUrl` 是否包含 `model.mify.ai.srv`。

---

## 5. 对照实验：取消 UPSTREAM_OPENAI_URL 再打一遍

```bash
unset UPSTREAM_OPENAI_URL

curl -i -N \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TEST" \
  http://127.0.0.1:4040/v1/chat/completions \
  -d '{
    "model": "xiaomi/mimo-v2-flash",
    "messages": [
      { "role": "user", "content": "ping-compare" }
    ],
    "stream": false
  }'
```

预期：如果没有 override，proxy 会走默认 OpenAI 上游（通常是 `https://api.openai.com`）。

---

## 6. 不看 UI，直接检查落盘的 `.lhar`

Context Lens 默认把会话写到：

```bash
~/.context-lens/data/
```

### 6.1 找最新的 LHAR 文件

```bash
ls -lt ~/.context-lens/data/*.lhar | head -n 5
```

把最新文件名记为 `<LATEST.lhar>`。

### 6.2 看每条 entry 的 URL / 状态码

```bash
jq -r '.entries[] | [.timestamp, .http.url, (.http.status_code|tostring)] | @tsv' \
  ~/.context-lens/data/<LATEST.lhar> \
  | tail -n 50
```

你关心的是：`.http.url` 是否指向 `http://model.mify.ai.srv/v1/...`。

---

## 7. 常见问题与排查

### Q1：UI 里还是显示旧的上游 URL（比如 opencode.ai 或 api.openai.com）

通常原因：`UPSTREAM_OPENAI_URL` 没进入 proxy 进程。

解决：停止后台并带环境变量重启：

```bash
context-lens background stop
UPSTREAM_OPENAI_URL="http://model.mify.ai.srv/v1" context-lens background start
```

### Q2：上游 URL 看起来像 `.../v1/v1/...`（双 /v1）

这是 URL 拼接/规范化问题（baseURL 已含 `/v1`，请求路径又带 `/v1/...`）。

临时绕过：把 `UPSTREAM_OPENAI_URL` 去掉 `/v1`：

```bash
context-lens background stop
UPSTREAM_OPENAI_URL="http://model.mify.ai.srv" context-lens background start
```

### Q3：opencode 场景 raw body 是 null，看不到 request/response

你贴的情况属于 `text/event-stream`（SSE streaming）。0.6.1 版本对 SSE raw body 可能不落盘。

这份验证文档的目的就是：用 `curl + stream:false` 绕开 SSE，确认代理转发链路与记录的 URL 是否正确。

---

## 8. 反馈我哪些信息可以快速定位

如果结果仍然“不对”，请贴（可脱敏 Authorization）：

1) `context-lens background status` 输出
2) 最新 LHAR 的 URL 列表（第 6.2 的 jq 输出）
3) 你执行的 curl 命令（确保包含 URL 路径）
