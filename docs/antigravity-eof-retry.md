# Antigravity EOF 传输层重试与容错修复

## 问题背景

### 现象

CProxyProxy 管理面板中大量 Antigravity 账号上报 `Post "https://cloudcode-pa.googleapis.com/v1internal:streamGenerateContent?alt=sse": EOF` 错误，导致请求失败，用户无法正常使用 AI 服务。

### 根因分析

Google Cloud Code (Antigravity 上游) 的 TCP 连接存在两类不稳定因素：

1. **连接被远端强制关闭（EOF）**
   - Google 服务端在高负载或内部调度时主动断开 TCP 连接
   - 连接池中空闲连接超时被回收，但客户端尚未感知
   - 表现为 `httpClient.Do()` 返回 `*url.Error` 包裹的 `io.EOF`

2. **Token 刷新链路同样脆弱**
   - OAuth token 刷新请求 `POST oauth2.googleapis.com/token` 同样经过公网 TCP
   - 刷新链路无重试，单次 EOF 即可导致整个 Auth 进入错误状态
   - 多个 Auth 共享同一刷新路径，单点故障放大

3. **WSL2 NAT 环境加剧**
   - WSL2 的 NAT 代理层会对长连接进行超时断开
   - TCP keepalive 在 WSL2 NAT 下的穿透不可靠
   - 管理 API/OAuth 客户端默认使用 HTTP/2，在 WSL NAT 下流式 EOF 概率更高

### 为什么不在 HTTP 响应层重试

EOF 发生在 `httpClient.Do()` 阶段（传输层），此时尚未收到 HTTP 状态码和响应体。原有的重试逻辑全部依赖 HTTP 状态码（429/503），对传输层错误仅做了 fallback base URL 切换，不进行同 URL 重试——这意味着如果只有一个 base URL 且遇到 EOF，请求直接失败。

## 解决方案

### 1. 传输层错误自动重试

在所有 `httpClient.Do()` 返回非 context 错误的路径上，新增：

```
请求失败 (EOF/reset/pipe)
  → 关闭空闲连接 (CloseIdleConnections)  // 清理失效连接池
  → 尝试下一个 base URL                     // 优先切换端点
  → 若无可用 base URL，同 URL 重试          // 指数退避: 500ms/1s/1.5s/2s
  → 达到 attempts 上限后返回错误
```

**涉及 3 个执行路径**：
- `executeNonStreaming` — 非流式请求
- `executeStreaming` — 流式 SSE 请求
- `executeLoadCodeAssist` — Code Assist 请求

**重试策略**：
```
attempt 0 → 500ms
attempt 1 → 1s
attempt 2 → 1.5s
attempt 3+ → 2s (cap)
```

### 2. Token 刷新链路硬化

将单次 `httpClient.Do()` 改为带退避的重试循环：

```
for attempt := 0; attempt < attempts; attempt++ {
    resp, err := httpClient.Do(refreshReq)
    if err == nil { break }
    if isContextError(err) || attempt+1 >= attempts { return err }
    CloseIdleConnections(client)
    sleep(transportRetryDelay(attempt))
}
```

同时给刷新请求设置 `httpReq.Close = true`，禁止连接复用，确保每次重试使用全新 TCP 连接。

### 3. MODEL_CAPACITY_EXHAUSTED 检测

Google Cloud Code 的容量耗尽有两种表达方式：

| 方式 | 响应码 | 载体 |
|------|--------|------|
| `"no capacity available"` | 503 | `body` 中的错误消息文本 |
| `MODEL_CAPACITY_EXHAUSTED` | 503 | `body` 中 JSON 的 `error.details[].reason` 字段 |

原有逻辑只检测第一种。新增 JSON reason 解析，从 `error.details.#.reason` 中匹配 `MODEL_CAPACITY_EXHAUSTED`，使 Google 原生格式的容量耗尽也能触发重试。

### 4. 空流 502 响应

当上游 SSE 流在发送任何数据前关闭（`dataChan` 收到 nil chunk 且累积 data 为空），不再发送空的 SSE 头部给客户端，而是返回 **502 Bad Gateway**：

```
h.WriteErrorResponse(c, &interfaces.ErrorMessage{
    StatusCode: http.StatusBadGateway,
    Error:      fmt.Errorf("upstream returned empty stream"),
})
cliCancel(fmt.Errorf("upstream returned empty stream"))
```

这使客户端（如 Claude Code）能够识别为可重试错误，而非收到 `Stream ended without receiving any events` 的不可恢复错误。

### 5. 管理连接强制 HTTP/1.1

WSL2 NAT 环境下 HTTP/2 的流复用与 NAT 超时策略存在兼容性问题。对管理 API 和 OAuth 客户端强制使用 HTTP/1.1：

```go
// 管理 API HTTP 客户端
transport.ForceAttemptHTTP2 = false

// OAuth 刷新 HTTP 客户端
transport.TLSClientConfig.NextProtos = []string{"http/1.1"}
```

HTTP/1.1 的短连接模型在 WSL NAT 下具有更可预测的超时行为。

## 文件变更

| 文件 | 变更 |
|------|------|
| `internal/runtime/executor/antigravity_executor.go` | 传输重试逻辑、MODEL_CAPACITY_EXHAUSTED 检测、token 刷新硬化、`antigravityCloseIdleConnections`、`antigravityTransportRetryDelay` |
| `internal/api/handlers/management/api_tools.go` | 管理 API HTTP 客户端强制 HTTP/1.1 |
| `internal/auth/antigravity/auth.go` | OAuth 刷新客户端强制 HTTP/1.1、WSL NAT 兼容 |
| `sdk/auth/filestore.go` | 文件存储层 HTTP 客户端 HTTP/1.1 适配 |
| `sdk/api/handlers/claude/code_handlers.go` | 空流 502 响应 |

## 验证

```bash
gofmt -w . && go build -o cli-proxy-api ./cmd/server && go test ./internal/runtime/executor/...
```

所有 executor 测试通过，编译无错误。

## 相关提交

- `fix(antigravity): retry token and transport EOF paths` — 传输重试 + token 刷新硬化
- `fix(management): force HTTP/1.1 on management API/OAuth client connections` — WSL NAT 兼容
- `fix(antigravity): add MODEL_CAPACITY_EXHAUSTED retry detection and empty stream 502 response` — 容量耗尽检测 + 空流修复
