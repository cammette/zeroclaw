# ZeroClaw Gateway API 文档

本文档描述 ZeroClaw Gateway 提供的 REST 和 WebSocket API 接口。

## 认证

大多数 API 端点需要 Bearer Token 认证。通过以下方式之一提供 token：

1. `Authorization: Bearer <token>` HTTP 头
2. WebSocket 协议子协议：`Sec-WebSocket-Protocol: bearer.<token>`
3. 查询参数：`?token=<token>`

---

## 1. 系统状态模块

### GET /api/status

获取系统状态概览。

**认证**: 需要

**响应示例**:
```json
{
  "provider": "openrouter",
  "model": "anthropic/claude-sonnet-4",
  "temperature": 0.7,
  "uptime_seconds": 3600,
  "gateway_port": 3000,
  "locale": "en-US",
  "memory_backend": "qdrant",
  "paired": true,
  "channels": {
    "telegram": true,
    "discord": false
  },
  "health": {
    // 健康检查详情
  }
}
```

### GET /api/health

获取组件健康快照。

**认证**: 需要

**响应示例**:
```json
{
  "health": {
    // 组件健康状态
  }
}
```

### GET /api/config

获取当前配置（敏感信息已脱敏）。

**认证**: 需要

**响应示例**:
```json
{
  "format": "toml",
  "content": "# 配置内容，敏感字段显示为 ***MASKED***"
}
```

### PUT /api/config

更新配置。

**认证**: 需要

**请求体**: TOML 格式的配置字符串

**响应**:
```json
{
  "status": "ok"
}
```

**错误**:
- `400 Bad Request`: 无效的 TOML 格式或配置验证失败
- `500 Internal Server Error`: 保存配置失败

---

## 2. 配置管理模块

### GET /api/tools

列出已注册的工具规格。

**认证**: 需要

**响应示例**:
```json
{
  "tools": [
    {
      "name": "shell",
      "description": "Execute shell commands",
      "parameters": {
        "type": "object",
        "properties": {
          "command": { "type": "string" }
        }
      }
    }
  ]
}
```

### GET /api/cli-tools

列出发现的 CLI 工具。

**认证**: 需要

**响应示例**:
```json
{
  "cli_tools": [
    {
      "name": "git",
      "path": "/usr/bin/git",
      "description": "version control system"
    }
  ]
}
```

### GET /api/cost

获取成本摘要。

**认证**: 需要

**响应示例**:
```json
{
  "cost": {
    "session_cost_usd": 0.0125,
    "daily_cost_usd": 0.45,
    "monthly_cost_usd": 13.5,
    "total_tokens": 15000,
    "request_count": 42,
    "by_model": {
      "anthropic/claude-sonnet-4": 0.0125
    }
  }
}
```

### POST /api/doctor

运行诊断检查。

**认证**: 需要

**响应示例**:
```json
{
  "results": [
    {
      "name": "provider_connection",
      "severity": "ok",
      "message": "Provider connection successful"
    }
  ],
  "summary": {
    "ok": 5,
    "warnings": 1,
    "errors": 0
  }
}
```

---

## 3. 定时任务模块

### GET /api/cron

列出所有定时任务。

**认证**: 需要

**响应示例**:
```json
{
  "jobs": [
    {
      "id": "job-uuid",
      "name": "daily-summary",
      "schedule": "0 9 * * *",
      "job_type": "agent",
      "prompt": "summarize yesterday's logs",
      "enabled": true,
      "delivery": {
        "mode": "announce",
        "channel": "discord",
        "to": "1234567890"
      }
    }
  ]
}
```

### POST /api/cron

添加新的定时任务。

**认证**: 需要

**请求体**:
```json
{
  "name": "job-name",
  "schedule": "*/5 * * * *",
  "job_type": "agent",
  "prompt": "执行任务的内容",
  "session_target": "session-id",
  "model": "anthropic/claude-sonnet-4",
  "allowed_tools": ["shell"],
  "delete_after_run": false,
  "delivery": {
    "mode": "announce",
    "channel": "discord",
    "to": "channel-id"
  }
}
```

或 Shell 任务：
```json
{
  "name": "backup",
  "schedule": "0 2 * * *",
  "command": "/path/to/backup.sh",
  "delivery": {
    "mode": "announce",
    "channel": "discord",
    "to": "channel-id"
  }
}
```

**响应**:
```json
{
  "status": "ok",
  "job": { /* 完整的任务对象 */ }
}
```

### GET /api/cron/:id/runs

获取定时任务的运行记录。

**认证**: 需要

**查询参数**:
- `limit` (可选, 默认 20, 范围 1-100): 返回的记录数

**响应示例**:
```json
{
  "runs": [
    {
      "id": "run-uuid",
      "job_id": "job-uuid",
      "started_at": "2026-04-27T09:00:00Z",
      "finished_at": "2026-04-27T09:00:15Z",
      "status": "success",
      "output": "任务输出",
      "duration_ms": 15000
    }
  ]
}
```

### PATCH /api/cron/:id

更新定时任务。

**认证**: 需要

**请求体**:
```json
{
  "name": "新名称",
  "schedule": "新的 cron 表达式",
  "command": "新命令或 prompt"
}
```

### DELETE /api/cron/:id

删除定时任务。

**认证**: 需要

**响应**:
```json
{
  "status": "ok"
}
```

### GET /api/cron/settings

获取定时任务子系统设置。

**认证**: 需要

**响应示例**:
```json
{
  "enabled": true,
  "catch_up_on_startup": true,
  "max_run_history": 100
}
```

### PATCH /api/cron/settings

更新定时任务子系统设置。

**认证**: 需要

**请求体**:
```json
{
  "enabled": true,
  "catch_up_on_startup": true,
  "max_run_history": 100
}
```

**响应**:
```json
{
  "status": "ok",
  "enabled": true,
  "catch_up_on_startup": true,
  "max_run_history": 100
}
```

---

## 4. 内存模块

### GET /api/memory

列出或搜索内存条目。

**认证**: 需要

**查询参数**:
- `query` (可选): 搜索查询
- `category` (可选): 分类筛选 (`core`, `daily`, `conversation`)
- `since` (可选): RFC 3339 时间，筛选此时间之后的条目
- `until` (可选): RFC 3339 时间，筛选此时间之前的条目

**响应示例**:
```json
{
  "entries": [
    {
      "key": "memory-key",
      "content": "内存内容",
      "category": "core",
      "created_at": "2026-04-27T10:00:00Z",
      "session_id": null
    }
  ]
}
```

### POST /api/memory

存储内存条目。

**认证**: 需要

**请求体**:
```json
{
  "key": "unique-key",
  "content": "要存储的内容",
  "category": "core"
}
```

**响应**:
```json
{
  "status": "ok"
}
```

### DELETE /api/memory/:key

删除内存条目。

**认证**: 需要

**响应示例**:
```json
{
  "status": "ok",
  "deleted": true
}
```

---

## 5. 集成模块

### GET /api/integrations

列出所有集成及其状态。

**认证**: 需要

**响应示例**:
```json
{
  "integrations": [
    {
      "name": "composio",
      "description": "Composio integration",
      "category": "tools",
      "status": "active"
    },
    {
      "name": "github",
      "description": "GitHub integration",
      "category": "tools",
      "status": "not_configured"
    }
  ]
}
```

### GET /api/integrations/settings

获取各集成的设置。

**认证**: 需要

**响应示例**:
```json
{
  "settings": {
    "composio": {
      "enabled": true,
      "category": "tools",
      "status": "active"
    },
    "github": {
      "enabled": false,
      "category": "tools",
      "status": "not_configured"
    }
  }
}
```

---

## 6. 会话管理模块

### GET /api/sessions

列出所有 Gateway 会话。

**认证**: 需要

**响应示例**:
```json
{
  "sessions": [
    {
      "session_id": "session-uuid",
      "name": "开发调试会话",
      "created_at": "2026-04-27T10:00:00Z",
      "last_activity": "2026-04-27T11:30:00Z",
      "message_count": 42
    }
  ]
}
```

### GET /api/sessions/running

列出当前正在运行的会话。

**认证**: 需要

**响应示例**:
```json
{
  "sessions": [
    {
      "session_id": "session-uuid",
      "created_at": "2026-04-27T10:00:00Z",
      "last_activity": "2026-04-27T11:30:00Z",
      "message_count": 42
    }
  ]
}
```

### GET /api/sessions/:id/messages

获取会话的消息历史。

**认证**: 需要

**响应示例**:
```json
{
  "session_id": "session-uuid",
  "messages": [
    {
      "role": "user",
      "content": "用户消息"
    },
    {
      "role": "assistant",
      "content": "助手回复"
    }
  ],
  "session_persistence": true
}
```

### GET /api/sessions/:id/state

获取会话状态。

**认证**: 需要

**响应示例**:
```json
{
  "session_id": "session-uuid",
  "state": "running",
  "turn_id": "turn-uuid",
  "turn_started_at": "2026-04-27T11:30:00Z"
}
```

### PUT /api/sessions/:id

重命名会话。

**认证**: 需要

**请求体**:
```json
{
  "name": "新会话名称"
}
```

**响应**:
```json
{
  "session_id": "session-uuid",
  "name": "新会话名称"
}
```

### DELETE /api/sessions/:id

删除会话。

**认证**: 需要

**响应**:
```json
{
  "deleted": true,
  "session_id": "session-uuid"
}
```

### POST /api/sessions/:id/abort

取消正在运行的会话响应。

**认证**: 需要

**响应**:
```json
{
  "status": "aborted"
}
```

或
```json
{
  "status": "no_active_response"
}
```

---

## 7. 设备配对模块

### POST /api/pairing/initiate

发起新的配对会话。

**认证**: 需要

**响应示例**:
```json
{
  "pairing_code": "ABC-123",
  "message": "New pairing code generated"
}
```

### POST /api/pair

提交配对代码进行设备配对。

**认证**: 不需要（但受速率限制）

**请求体**:
```json
{
  "code": "ABC-123",
  "device_name": "我的手机",
  "device_type": "mobile"
}
```

**响应**:
```json
{
  "token": "zc_xxx...",
  "message": "Pairing successful"
}
```

**错误**:
- `400 Bad Request`: 无效或过期的配对代码
- `429 Too Many Requests`: 尝试次数过多

### GET /api/devices

列出已配对的设备。

**认证**: 需要

**响应示例**:
```json
{
  "devices": [
    {
      "id": "device-uuid",
      "name": "我的手机",
      "device_type": "mobile",
      "paired_at": "2026-04-27T10:00:00Z",
      "last_seen": "2026-04-27T11:30:00Z",
      "ip_address": "192.168.1.100"
    }
  ],
  "count": 1
}
```

### DELETE /api/devices/:id

撤销已配对的设备。

**认证**: 需要

**响应**:
```json
{
  "message": "Device revoked",
  "device_id": "device-uuid"
}
```

### POST /api/devices/:id/token/rotate

轮换设备的 token。

**认证**: 需要

**响应**:
```json
{
  "device_id": "device-uuid",
  "pairing_code": "NEW-CODE",
  "message": "Use this code to re-pair the device"
}
```

---

## 8. Live Canvas 模块

### GET /api/canvas

列出所有活跃的 canvas。

**认证**: 需要

**响应示例**:
```json
{
  "canvases": ["canvas-1", "canvas-2"]
}
```

### GET /api/canvas/:id

获取 canvas 当前内容。

**认证**: 需要

**响应示例**:
```json
{
  "canvas_id": "canvas-1",
  "frame": {
    "content_type": "html",
    "content": "<html>...</html>",
    "created_at": "2026-04-27T10:00:00Z"
  }
}
```

### GET /api/canvas/:id/history

获取 canvas 帧历史。

**认证**: 需要

**响应示例**:
```json
{
  "canvas_id": "canvas-1",
  "frames": [
    {
      "content_type": "html",
      "content": "<html>...</html>",
      "created_at": "2026-04-27T10:00:00Z"
    }
  ]
}
```

### POST /api/canvas/:id

向 canvas 推送内容。

**认证**: 需要

**请求体**:
```json
{
  "content_type": "html",
  "content": "<html>...</html>"
}
```

**响应**:
```json
{
  "canvas_id": "canvas-1",
  "frame": {
    "content_type": "html",
    "content": "<html>...</html>",
    "created_at": "2026-04-27T10:00:00Z"
  }
}
```

**错误**:
- `400 Bad Request`: 无效的 content_type
- `413 Payload Too Large`: 内容超过最大限制
- `429 Too Many Requests`: 达到最大 canvas 数量

### DELETE /api/canvas/:id

清空 canvas。

**认证**: 需要

**响应**:
```json
{
  "canvas_id": "canvas-1",
  "status": "cleared"
}
```

---

## 9. WebAuthn 模块 (需 `webauthn` feature)

### POST /api/webauthn/register/start

开始 WebAuthn 注册流程。

**认证**: 需要

**请求体**:
```json
{
  "user_id": "user-123",
  "user_name": "张三"
}
```

**响应**:
```json
{
  "public_key": {
    "challenge": "...",
    "rp": { "name": "ZeroClaw" },
    "user": {
      "id": "...",
      "name": "张三"
    },
    "pubKeyCredParams": [
      { "type": "public-key", "alg": -7 }
    ]
  }
}
```

### POST /api/webauthn/register/finish

完成 WebAuthn 注册。

**认证**: 需要

**请求体**:
```json
{
  "challenge": "...",
  "credential_id": "...",
  "client_data_json": "...",
  "authenticator_data": "..."
}
```

**响应**:
```json
{
  "credential_id": "...",
  "label": "我的安全密钥",
  "registered_at": "2026-04-27T10:00:00Z"
}
```

### POST /api/webauthn/auth/start

开始 WebAuthn 认证流程。

**认证**: 需要

**请求体**:
```json
{
  "user_id": "user-123"
}
```

**响应**:
```json
{
  "public_key": {
    "challenge": "...",
    "allowCredentials": [
      {
        "type": "public-key",
        "id": "..."
      }
    ]
  }
}
```

### POST /api/webauthn/auth/finish

完成 WebAuthn 认证。

**认证**: 需要

**请求体**:
```json
{
  "challenge": "...",
  "credential_id": "...",
  "client_data_json": "...",
  "authenticator_data": "...",
  "signature": "..."
}
```

**响应**:
```json
{
  "status": "authenticated"
}
```

### GET /api/webauthn/credentials?user_id=...

列出用户的凭证。

**认证**: 需要

**响应示例**:
```json
{
  "credentials": [
    {
      "credential_id": "...",
      "label": "我的安全密钥",
      "registered_at": "2026-04-27T10:00:00Z",
      "sign_count": 42
    }
  ]
}
```

### DELETE /api/webauthn/credentials/:id?user_id=...

删除凭证。

**认证**: 需要

**响应**:
```json
{
  "status": "deleted"
}
```

---

## 10. 插件模块 (需 `plugins-wasm` feature)

### GET /api/plugins

列出已加载的插件。

**认证**: 需要

**响应示例**:
```json
{
  "plugins_enabled": true,
  "plugins_dir": "~/.config/zeroclaw/plugins",
  "plugins": [
    {
      "name": "custom-tool",
      "version": "1.0.0",
      "description": "自定义工具插件",
      "capabilities": ["tool"],
      "loaded": true
    }
  ]
}
```

---

## 11. WebSocket 接口

### WS /ws/chat

实时聊天 WebSocket 连接。

**查询参数**:
- `session_id` (可选): 恢复或创建会话 ID
- `name` (可选): 会话的可读名称
- `token` (可选): Bearer token（替代 Authorization 头）

**协议**:

客户端 → 服务器:
```json
{"type":"message","content":"你好"}
```

或连接参数（可选）:
```json
{"type":"connect","session_id":"xxx","device_name":"我的设备","capabilities":[]}
```

服务器 → 客户端:
```json
{"type":"session_start","session_id":"xxx","name":"会话名","resumed":true,"message_count":10}
```

```json
{"type":"chunk","content":"Hi! "}
```

```json
{"type":"tool_call","name":"shell","args":{"command":"ls"}}
```

```json
{"type":"tool_result","name":"shell","output":"..."}
```

```json
{"type":"done","full_response":"完整回复"}
```

```json
{"type":"error","message":"错误信息","code":"ERROR_CODE"}
```

```json
{"type":"aborted"}
```

**认证方式**:
1. `Authorization: Bearer <token>` 头
2. `Sec-WebSocket-Protocol: bearer.<token>` 子协议
3. `?token=<token>` 查询参数

---

### WS /ws/canvas/:id

Canvas 实时更新 WebSocket。

**认证**: 需要

**协议**:

服务器 → 客户端:
```json
{"type":"connected","canvas_id":"canvas-1"}
```

```json
{"type":"frame","canvas_id":"canvas-1","frame":{"content_type":"html","content":"..."}}
```

```json
{"type":"lagged","canvas_id":"canvas-1","missed_frames":5}
```

```json
{"type":"error","error":"错误信息"}
```

---

### WS /ws/nodes

节点发现和能力广播 WebSocket。

**查询参数**:
- `token` (可选): 节点认证 token

**协议**:

节点 → 网关:
```json
{"type":"register","node_id":"phone-1","capabilities":[{"name":"camera.snap","description":"拍照","parameters":{}}]}
```

```json
{"type":"result","call_id":"uuid","success":true,"output":"...","error":null}
```

网关 → 节点:
```json
{"type":"registered","node_id":"phone-1","capabilities_count":1}
```

```json
{"type":"invoke","call_id":"uuid","capability":"camera.snap","args":{}}
```

**认证方式**:
1. 节点专用 token（如果配置了 `nodes.auth_token`）
2. 或使用配对 token

---

## 12. 语音双工模块 (需 `gateway-voice-duplex` feature)

在 `/ws/chat` WebSocket 中支持以下语音事件：

### 客户端 → 服务器

```json
{"type":"speech_start"}
```

```json
{"type":"speech_end","transcript":"识别的文本"}
```

```json
{"type":"barge_in"}
```

### 服务器 → 客户端

```json
{"type":"tts_chunk","audio_b64":"base64编码的音频数据","format":"mp3"}
```

```json
{"type":"tts_cancel"}
```

---

## 13. Webhook

### POST /hooks/claude-code

接收来自 Claude Code 的 HTTP hook 事件（不需要认证）。

**请求体**:
```json
{
  "session_id": "session-uuid",
  "event_type": "tool_call",
  "tool_name": "bash",
  "summary": {
    "success": true,
    "output": "..."
  }
}
```

**响应**:
```json
{
  "ok": true
}
```

---

## 错误码

| 错误码 | 描述 |
|--------|------|
| `UNAUTHORIZED` | 未授权，需要有效的 bearer token |
| `INVALID_JSON` | 无效的 JSON |
| `EMPTY_CONTENT` | 消息内容不能为空 |
| `UNKNOWN_MESSAGE_TYPE` | 不支持的消息类型 |
| `SESSION_BUSY` | 会话正忙 |
| `AGENT_INIT_FAILED` | Agent 初始化失败 |
| `AUTH_ERROR` | 认证错误（API Key 问题） |
| `PROVIDER_ERROR` | 提供商错误 |
| `AGENT_ERROR` | Agent 错误 |
| `invalid_event_direction` | 错误的事件方向（语音双工） |
| `AGENT_INIT_FAILED` | Agent 初始化失败 |

---

## 注意事项

1. **速率限制**: API 受速率限制保护，超限将返回 `429 Too Many Requests`
2. **请求大小限制**: 最大请求体大小为 64KB
3. **请求超时**: 默认请求超时为 30 秒（可通过 `ZEROCLAW_GATEWAY_TIMEOUT_SECS` 环境变量调整）
4. **敏感信息**: 获取配置时，敏感信息（如 API keys）会被脱敏为 `***MASKED***`
