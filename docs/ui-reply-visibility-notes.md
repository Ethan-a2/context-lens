# UI 回复可见性 / raw body 记录 — 关键结论与复现信息

日期：2026-03-02

## 现象
- curl 请求能正常得到模型回复（HTTP 200）。
- `.lhar` 里能看到 `raw.response_body`（在 `CONTEXT_LENS_PRIVACY=full` 时）。
- 但 UI（`/api/entries/:id/detail` 与 MessagesTab）看不到 assistant 回复文本，只能看到 user。

## 已验证：转发到自定义 OpenAI-compatible 上游生效
环境变量覆盖上游（注意：需要在 proxy 启动时就存在）：

```bash
context-lens background stop
CONTEXT_LENS_PRIVACY=full UPSTREAM_OPENAI_URL="http://model.mify.ai.srv" context-lens background start
```

发送非流式 Chat Completions：

```bash
curl -i -N \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <REDACTED>" \
  http://127.0.0.1:4040/v1/chat/completions \
  -d '{
    "model": "xiaomi/mimo-v2-flash",
    "messages": [
      { "role": "user", "content": "hi , 1+1=" }
    ],
    "stream": false
  }'
```

结果：
- HTTP 200
- entry 中 `targetUrl` / `http.url` 指向 `http://model.mify.ai.srv/v1/chat/completions`

## 关键原因：UI 只渲染 contextInfo.messages（请求侧），不渲染 response
`/api/entries/<id>/detail` 当前只返回：
- `{ contextInfo: ... }`

示例：

```bash
curl http://localhost:4041/api/entries/45/detail | jq
```

返回只包含 user message：

```json
{
  "contextInfo": {
    "provider": "openai",
    "apiFormat": "chat-completions",
    "model": "xiaomi/mimo-v2-flash",
    "messages": [
      { "role": "user", "content": "hi , 1+1=", "tokens": 33 }
    ]
  }
}
```

UI 侧：
- `ui/src/components/MessagesTab.vue` 会优先使用 entry detail 的 `contextInfo.messages`（若存在），否则退回 `entry.contextInfo.messages`。
- `parseContextInfo()` 仅从 request body 构造 `contextInfo`，因此通常不会包含 assistant 回复。

## raw body 是否落盘（LHAR）取决于隐私级别
- LHAR 里 `raw.request_body/response_body` 的写入由隐私级别控制：只有 `privacy === "full"` 才会写 raw body，否则为 `null`。
- 设置 `CONTEXT_LENS_PRIVACY=full` 后，LHAR 可见（示意）：

```json
"raw": {
  "request_body": { "model": "...", "messages": [ ... ], "stream": false },
  "response_body": {
    "choices": [
      {
        "message": {
          "role": "assistant",
          "content": "Hello! The answer is **2** ..."
        }
      }
    ]
  }
}
```

## 结论
- “UI 看不到回复”不是不支持 UI，而是：
  1) 后端 detail API 只提供 `contextInfo`（请求侧）
  2) `contextInfo.messages` 当前不包含 assistant（因为是从 request 解析的）
  3) UI Messages 只渲染 messages，不从 response/raw 中解析 assistant

## 后续修复方向（待选择）
1) **推荐**：在 ingest/store 阶段把非流式响应（chat-completions/responses）解析出 assistant 文本，并追加到 `contextInfo.messages`，使 UI Messages 自动显示。
2) 扩展 detail API（以及 UI）直接返回/显示 `response` 或 `raw.response_body`（需考虑隐私级别与体积）。
3) 若要支持流式（SSE），需要把 chunks 重组后再提取 assistant（复杂度更高）。
