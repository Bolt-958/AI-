# Hermes 基础命令与排错整理

本文档整理本次配置 Hermes Agent、Gateway、Feishu、hermes-web-ui 时用到的基础命令、请求示例、文件位置和常见报错含义。

## 1. 基础路径

| 项目 | 路径 / 地址 | 含义 |
| --- | --- | --- |
| Hermes 主目录 | `C:\Users\30806\AppData\Local\hermes` | 当前实际使用的 Hermes 配置目录 |
| 兼容路径 | `C:\Users\30806\.hermes` | 已指向 Hermes 主目录的兼容入口 |
| Web UI 日志 | `C:\Users\30806\.hermes-web-ui\server.log` | hermes-web-ui 启动、代理、报错日志 |
| Gateway 日志 | `C:\Users\30806\AppData\Local\hermes\logs\gateway.log` | default gateway 日志 |
| Gateway stdio 日志 | `C:\Users\30806\AppData\Local\hermes\logs\gateway-stdio.log` | gateway 标准输出/错误日志 |
| processor 配置目录 | `C:\Users\30806\AppData\Local\hermes\profiles\processor` | processor agent 的独立配置目录 |
| lookout 配置目录 | `C:\Users\30806\AppData\Local\hermes\profiles\lookout` | lookout agent 的独立配置目录 |

## 2. Profile / Agent 管理

### 查看所有 agent/profile

```powershell
hermes profile list
```

含义：

- 查看当前有哪些 profile。
- 查看每个 profile 使用的模型。
- 查看 gateway 是否运行。
- `◆` 表示当前默认/当前激活 profile。

### 使用指定 profile 执行一次对话

```powershell
hermes --profile processor -z "只回复 OK"
```

含义：

- 临时使用 `processor` 这个 agent。
- `-z` 表示执行一次非交互式请求。
- 如果返回正常，说明该 agent 的模型、base_url、api_key 基本可用。

### 切换当前默认 profile

```powershell
hermes profile use processor
```

含义：

- 把默认 profile 切换成 `processor`。
- 后续直接运行 `hermes` 时会默认使用它。

切回 default：

```powershell
hermes profile use default
```

## 3. Gateway 管理

### 查看当前 profile 的 gateway 状态

```powershell
hermes gateway status
```

含义：

- 查看当前默认 profile 的 gateway 是否注册、是否运行。
- 如果显示 `No gateway process detected`，说明当前默认 profile 没有运行中的 gateway。
- `Other profiles` 下面显示其他 profile 的 gateway 状态。

### 查看指定 profile 的 gateway 状态

```powershell
hermes --profile processor gateway status
```

含义：

- 专门查看 `processor` 的 gateway。
- 本次排查中 processor gateway 运行在 `127.0.0.1:8645`。

### 启动当前 profile 的 gateway

```powershell
hermes gateway start
```

### 启动指定 profile 的 gateway

```powershell
hermes --profile processor gateway start
```

### 重启指定 profile 的 gateway

```powershell
hermes --profile processor gateway restart
```

### 停止指定 profile 的 gateway

```powershell
hermes --profile processor gateway stop
```

## 4. 当前 Gateway 端口

本次修复后确认到的端口：

| Profile | Gateway 地址 | 状态 |
| --- | --- | --- |
| default | `http://127.0.0.1:8642` | running |
| lookout | `http://127.0.0.1:8644` | running |
| processor | `http://127.0.0.1:8645` | running |
| hermes-web-ui | `http://localhost:8648` | running |

查看端口占用：

```powershell
Get-NetTCPConnection -LocalPort 8648,8642,8644,8645 -ErrorAction SilentlyContinue |
  Select-Object LocalAddress,LocalPort,State,OwningProcess
```

查看对应进程：

```powershell
Get-Process -Id <PID>
```

## 5. hermes-web-ui 管理

### 启动 Web UI

```powershell
$env:HERMES_HOME='C:\Users\30806\AppData\Local\hermes'
$env:UPSTREAM='http://127.0.0.1:8645'
hermes-web-ui start
```

含义：

- `HERMES_HOME` 指定 Web UI 使用正确的 Hermes 配置目录。
- `UPSTREAM` 指定默认代理到哪个 gateway。
- 当前为了方便 processor 测试，默认 upstream 指向 `processor` 的 `8645`。

### 停止 Web UI

```powershell
hermes-web-ui stop
```

### Web UI 访问地址

```text
http://localhost:8648
```

带 token 的地址：

```text
http://localhost:8648/#/?token=f56904d3b78a9bb085ba075b2827b1c29bf06d41bdddba5b758fd4e68845f5e7
```

### 查看 Web UI 日志

```powershell
Get-Content -Path 'C:\Users\30806\.hermes-web-ui\server.log' -Tail 120
```

含义：

- 查看 Web UI 是否启动成功。
- 查看当前 upstream。
- 查看 Web UI 是否识别到所有 running gateway。
- 查看代理层错误。

## 6. Web UI API 测试请求

### 查看 Web UI 识别到的 gateway

```powershell
Invoke-WebRequest `
  -Uri 'http://127.0.0.1:8648/api/hermes/gateways' `
  -Headers @{Authorization='Bearer f56904d3b78a9bb085ba075b2827b1c29bf06d41bdddba5b758fd4e68845f5e7'} `
  -UseBasicParsing |
  Select-Object -ExpandProperty Content
```

正常结果应包含：

```json
{
  "gateways": [
    { "profile": "default", "port": 8642, "running": true },
    { "profile": "lookout", "port": 8644, "running": true },
    { "profile": "processor", "port": 8645, "running": true }
  ]
}
```

### 通过 Web UI 代理测试 processor 对话

```powershell
$headers=@{
  Authorization='Bearer f56904d3b78a9bb085ba075b2827b1c29bf06d41bdddba5b758fd4e68845f5e7'
  'Content-Type'='application/json'
  'x-hermes-profile'='processor'
}

$body='{
  "model":"processor",
  "messages":[
    {"role":"user","content":"只回复 OK"}
  ],
  "stream":false
}'

Invoke-WebRequest `
  -Uri 'http://127.0.0.1:8648/api/hermes/v1/chat/completions?profile=processor' `
  -Method Post `
  -Headers $headers `
  -Body $body `
  -UseBasicParsing `
  -TimeoutSec 150 |
  Select-Object -ExpandProperty Content
```

含义：

- 请求先进入 `hermes-web-ui`。
- Web UI 根据 `x-hermes-profile='processor'` 或 `?profile=processor` 转发到 processor gateway。
- 如果返回 `STATUS=200` 或 JSON completion，说明 Web UI 到 processor 的代理链路正常。

## 7. 直接测试 processor Gateway

### 读取 processor 的 API_SERVER_KEY

```powershell
Get-Content 'C:\Users\30806\AppData\Local\hermes\profiles\processor\.env'
```

其中：

```text
API_SERVER_KEY=...
```

是本地 processor gateway 的访问 key，不是模型供应商 key。

### 直接请求 processor gateway

```powershell
$key=(Get-Content 'C:\Users\30806\AppData\Local\hermes\profiles\processor\.env' |
  Where-Object {$_ -like 'API_SERVER_KEY=*'}).Split('=',2)[1]

$headers=@{
  Authorization="Bearer $key"
  'Content-Type'='application/json'
}

$body='{
  "model":"processor",
  "messages":[
    {"role":"user","content":"只回复 OK"}
  ],
  "stream":false
}'

Invoke-WebRequest `
  -Uri 'http://127.0.0.1:8645/v1/chat/completions' `
  -Method Post `
  -Headers $headers `
  -Body $body `
  -UseBasicParsing `
  -TimeoutSec 150 |
  Select-Object -ExpandProperty Content
```

含义：

- 绕过 Web UI，直接测试 processor gateway。
- 如果这里成功，但 Web UI 失败，说明问题在 Web UI 代理层。
- 如果这里失败，说明问题在 processor gateway、模型配置或 API key。

## 8. processor 模型配置

配置文件：

```text
C:\Users\30806\AppData\Local\hermes\profiles\processor\config.yaml
```

关键配置示例：

```yaml
model:
  default: gpt-5.4
  provider: custom
  base_url: https://codex.hualong.site/v1
  api_key: sk-...
```

含义：

- `default`：该 agent 默认使用的模型。
- `provider: custom`：使用自定义 OpenAI-compatible API。
- `base_url`：模型供应商接口地址。
- `api_key`：模型供应商 key。

注意：

- `config.yaml` 里的 `api_key` 是调用模型供应商用的。
- `.env` 里的 `API_SERVER_KEY` 是本地 gateway API server 的访问 key。
- 两者不是同一个东西。

## 9. Feishu 相关配置

processor 的 Feishu 配置位于：

```text
C:\Users\30806\AppData\Local\hermes\profiles\processor\.env
```

示例字段：

```text
FEISHU_APP_ID=...
FEISHU_APP_SECRET=...
FEISHU_DOMAIN=feishu
FEISHU_CONNECTION_MODE=websocket
FEISHU_ALLOW_ALL_USERS=true
FEISHU_GROUP_POLICY=open
```

含义：

- `FEISHU_APP_ID`：飞书机器人应用 ID。
- `FEISHU_APP_SECRET`：飞书机器人应用密钥。
- `FEISHU_CONNECTION_MODE=websocket`：使用 websocket 方式连接飞书。
- `FEISHU_ALLOW_ALL_USERS=true`：允许所有用户。
- `FEISHU_GROUP_POLICY=open`：群聊策略为开放。

## 10. 常见报错含义

### 1. `Invalid API key`

示例：

```text
Error: API Error 401: {"error":{"message":"Invalid API key","code":"invalid_api_key"}}
```

可能含义：

- 模型供应商 key 错误。
- Web UI 转发时使用了错误的 gateway key。
- Web UI 请求被转发到了错误的 profile/gateway。
- 本次 processor 问题中，表面看像 key 错，实际是 Web UI 代理层转发失败。

排查顺序：

1. 先用 `hermes --profile processor -z "只回复 OK"` 测试模型配置。
2. 再直接请求 `http://127.0.0.1:8645/v1/chat/completions`。
3. 最后请求 `http://127.0.0.1:8648/api/hermes/v1/chat/completions?profile=processor`。

### 2. `502 Bad Gateway`

含义：

- Web UI 代理层无法成功转发请求到 gateway。
- 不一定是模型 key 错。

本次发现的具体原因：

```text
NotSupportedError: expect header not supported
```

解释：

- Web UI 把客户端请求里的 `Expect` header 转发给 Node.js `fetch`。
- Node.js 的 undici fetch 不支持该 header。
- 请求还没发到 processor gateway 就失败。

已修复位置：

```text
C:\Users\30806\AppData\Local\hermes\node\node_modules\hermes-web-ui\dist\server\routes\hermes\proxy-handler.js
```

修复方式：

- 代理转发时过滤 `expect` header。
- 同时过滤 `content-length`，避免 body 重新序列化后长度不一致。

### 3. `Launched gateway via Scheduled Task ... but no process detected`

示例：

```text
Launched gateway via Scheduled Task 'Hermes_Gateway', but no process detected after 6s.
```

含义：

- Windows 计划任务已触发，但 gateway 进程没有成功保持运行。

排查日志：

```powershell
Get-Content 'C:\Users\30806\AppData\Local\hermes\logs\gateway.log' -Tail 120
Get-Content 'C:\Users\30806\AppData\Local\hermes\logs\gateway-stdio.log' -Tail 120
```

### 4. Web UI 只能看到一个 agent

可能原因：

- Web UI 读取了错误的 Hermes home。
- `C:\Users\30806\.hermes` 和 `C:\Users\30806\AppData\Local\hermes` 不一致。
- Web UI profile list 解析有问题。

当前处理方式：

- 设置 `HERMES_HOME=C:\Users\30806\AppData\Local\hermes`。
- 让 `C:\Users\30806\.hermes` 指向真实 Hermes 主目录。
- 修复 Web UI profile 解析后，当前能看到 `default`、`lookout`、`processor`。

## 11. 建议排查流程

当某个 agent 在 Web UI 里不能对话时，按这个顺序排查：

```powershell
hermes profile list
```

确认 agent 存在、模型正确。

```powershell
hermes --profile processor gateway status
```

确认 gateway 正在运行。

```powershell
hermes --profile processor -z "只回复 OK"
```

确认 agent 模型配置可用。

```powershell
Invoke-WebRequest -Uri 'http://127.0.0.1:8648/api/hermes/gateways' `
  -Headers @{Authorization='Bearer <WEB_UI_TOKEN>'} `
  -UseBasicParsing
```

确认 Web UI 能识别该 agent 的 gateway。

最后再通过 Web UI 页面测试对话。

## 12. 当前可用状态

截至本次修复完成：

```text
default   -> running -> 127.0.0.1:8642
lookout   -> running -> 127.0.0.1:8644
processor -> running -> 127.0.0.1:8645
Web UI    -> running -> localhost:8648
```

processor 已通过 Web UI 代理测试成功。
