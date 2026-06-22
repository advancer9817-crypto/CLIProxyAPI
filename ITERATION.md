# CPA 部署迭代记录

## 2026-05-27: cpa 一键管理 & CPA-Manager-Plus 作为默认前端

### 变更摘要
- CPA-Manager-Plus 前端重新构建，同步为 CPA 默认管理面板
- 创建 systemd user 服务 `cpa-proxy` 和 `cpa-manager-plus`
- 创建 `~/.local/bin/cpa` 一键管理命令

### 影响范围
| 模块 | 变更 |
|------|------|
| `static/management.html` | 重新构建的 CPAM 前端（2.9MB，最新版） |
| `config.yaml` | 无变更 |

### 新命令
```
cpa start    启动 CPA + CPAM
cpa stop     停止所有
cpa status   查看状态
cpa restart  重启
cpa logs     查看日志
cpa reload   重载配置
cpa enable   开机自启
cpa disable  取消自启
```

### 服务端口
- CPA Proxy:  :8317
- CPA Manager+: :18317

### 管理面板
- CPA 内置面板:  http://localhost:8317/management.html
- CPAM 独立面板: http://localhost:18317/management.html

### 回滚
1. 删除 `~/.local/bin/cpa`
2. 停止并禁用 systemd services: `systemctl --user disable cpa-proxy cpa-manager-plus`
3. 删除 `/home/advancer/.config/systemd/user/cpa-proxy.service` 和 `cpa-manager-plus.service`

## 2026-06-22: Antigravity EOF 传输层重试与容错修复

### 问题
CProxyProxy 管理面板中大量 Antigravity 账号报 `Post "cloudcode-pa.googleapis.com/v1internal:streamGenerateContent?alt=sse": EOF` 错误。

### 根因
Google Cloud Code 服务端在高负载或内部调度时主动断开 TCP 连接，WSL2 NAT 环境加剧了此问题。原有的重试逻辑只处理 HTTP 层面的 429/503 状态码，不对传输层错误（EOF/reset/broken pipe）做同 URL 重试。

### 解决思路
1. **传输层重试**：在 `httpClient.Do()` 返回错误后关闭空闲连接、切换 fallback base URL，若无可用端点则同 URL 带退避重试（500ms→2s cap）
2. **Token 刷新硬化**：OAuth 刷新改为带重试的循环，`httpReq.Close = true` 禁止连接复用
3. **MODEL_CAPACITY_EXHAUSTED 检测**：从 Google Cloud Code JSON 响应的 `error.details.#.reason` 中检测容量耗尽
4. **空流 502**：上游返回空 SSE 流时返回 502 Bad Gateway 给客户端，使其可重试
5. **HTTP/1.1 强制**：管理 API 和 OAuth 客户端禁用 HTTP/2，避免 WSL NAT 下的帧复用超时

### 文件变更
| 文件 | 变更 |
|------|------|
| `internal/runtime/executor/antigravity_executor.go` | 3 处传输重试 + token 刷新硬化 + MODEL_CAPACITY_EXHAUSTED + `antigravityTransportRetryDelay` / `antigravityCloseIdleConnections` |
| `internal/api/handlers/management/api_tools.go` | HTTP/1.1 强制 |
| `internal/auth/antigravity/auth.go` | HTTP/1.1 + WSL NAT 兼容 |
| `sdk/auth/filestore.go` | HTTP/1.1 适配 |
| `sdk/api/handlers/claude/code_handlers.go` | 空流 502 |

### 详细文档
见 `docs/antigravity-eof-retry.md`

### CPA-Manager-Plus 前端同步
合并 upstream v1.4.2→v1.8.0（269 文件），新增 Usage Analytics、插件商店、配额冷却自动化、Codex reauth、账户处理策略、ECharts、增强监控、国际化更新、Antigravity 订阅管理。

## 2026-05-29: Antigravity 代理稳定性修复与官方更新准备

### 变更摘要
- 修复管理 API 工具请求在 `proxy-url` 为空时未继承 `HTTP_PROXY`/`HTTPS_PROXY` 的行为。
- 为 Antigravity OAuth `userinfo` 与 `loadCodeAssist` 请求增加最多 3 次重试，降低临时 EOF 对登录和刷新流程的影响。
- 将 Antigravity fallback User-Agent 从 `antigravity/1.21.9` 更新为 `antigravity/2.0.0`，避免 updater 不可达时继续使用已不支持版本。
- 将本地管理密钥文件 `.current_admin_key` 加入 `.gitignore`，避免误提交。

### 修复
- 修复 `Post "https://cloudcode-pa.googleapis.com/v1internal:streamGenerateContent?alt=sse": EOF` 高频出现时，CPA 对 Antigravity/Claude 路径的代理和版本兜底不稳定问题。
- 保留 Clash 侧 `cloudcode-pa.googleapis.com` 与 `cloudcode-pa.sandbox.googleapis.com` 走 `OpenAI` 组的运行路径；不再放通 Antigravity updater 专用规则，避免触发旧版本提示。

### 测试结果
- `go test ./internal/misc ./internal/runtime/executor`: 2/2 packages pass；`internal/misc` 为 `[no test files]`；`internal/runtime/executor` 为 `ok`。
- `go test ./...`: 104/104 packages pass；51 packages `ok`，53 packages `[no test files]`，0 failures。
- 运行态验证：`cpa-proxy.service` 重启成功；重启后 `/v1/responses` 返回 `200`；`cloudcode-pa.googleapis.com` 通过 `172.20.0.1:7890` 代理返回预期根路径 `404`，确认 CONNECT/TLS 正常。

## 2026-05-29: Antigravity EOF 剩余路径硬化

### 变更摘要
- Antigravity OAuth 授权码换 token、access token refresh、管理端 Antigravity token refresh、auth 文件导入 token refresh 均改为传输错误可重试。
- Antigravity 主请求在连接建立或请求发送阶段遇到 EOF/socket close 时，先关闭 idle 连接，再按 `request-retry` 重建请求并重试。
- 管理 API 工具文档同步为 `proxy-url` 为空时继承 `HTTP_PROXY`/`HTTPS_PROXY`，与运行逻辑一致。

### 修复
- 修复 `Post "https://oauth2.googleapis.com/token": EOF` 在 token 交换、token refresh、管理端 token 替换和 auth 文件刷新路径中单次失败直接返回的问题。
- 修复 `Post "https://cloudcode-pa.googleapis.com/v1internal:streamGenerateContent?alt=sse": EOF` 在 Antigravity 请求发出前或建连阶段失败时没有进入同一轮传输重试的问题。
- 修复代理侧半关闭连接可能被继续复用的问题；每次可重试传输错误后显式关闭 idle 连接。

### 测试结果
- `go test ./internal/auth/antigravity ./internal/runtime/executor ./internal/api/handlers/management ./sdk/auth`: 4/4 packages pass。
- `go test ./...`: 104/104 packages pass，0 failures。
- `go build -o /home/advancer/project/CLIProxyAPI/cli-proxy-api ./cmd/server`: build pass。
- 运行态验证：`cpa-proxy.service` 重启成功，PID 143678，`HTTP_PROXY`/`HTTPS_PROXY` 均为 `http://172.20.0.1:7890`。
- POST 探测：`oauth2.googleapis.com/token` 5/5 次返回预期 `401` 且 0 EOF；`cloudcode-pa.googleapis.com/v1internal:streamGenerateContent?alt=sse` 5/5 次返回预期 `401` 且 0 EOF。
## [2026-06-12 15:35] Fix OAuth token refresh HTTP/2 stream errors (EOF)

In WSL2 / NAT proxy network environments, Google's OAuth2 endpoints enforce HTTP/2 when clients advertise support. Local proxy tools like Clash can prematurely tear down idle HTTP/2 stream connections, leading to transient 'HTTP/2 stream was not closed cleanly... EOF' errors.

- **Fix**: Modified `internal/api/handlers/management/api_tools.go` to force HTTP/1.1 (by disabling HTTP/2 upgrade via empty `TLSNextProto` and enforcing `NextProtos = []string{"http/1.1"}` on TLS client configuration) for all outbound API/token refresh calls going through `apiCallTransport` or `buildProxyTransport`. This matches the successful workaround implemented earlier in the Antigravity Executor.
- **Verification**: Built and verified the compilation, successfully tested model queries through proxy, and confirmed zero HTTP/2 stream EOF errors on repeated tokens refreshes.
