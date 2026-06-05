# Hermes 基础命令

本文档整理本次配置 Hermes Agent、Gateway、Feishu、hermes-web-ui 时用到的基础命令。

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

- 专门查看 特定（`processor`） 的 gateway。

### 启动当前 profile 的 gateway

```powershell
hermes gateway start
```

### 启动指定 profile 的 gateway

```powershell
hermes --profile processor gateway start
#也可以直接输入'名字'即可进入指定agent
processor
```

### 重启和停止指定 profile 的 gateway

```powershell
processor gateway restart
processor gateway stop
```

## 4. 查看Gateway 端口相关信息

我电脑上确认到的端口：

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

### 启动与停止 Web UI

```powershell
hermes-web-ui start
hermes-web-ui stop
```

### Web UI 访问地址

```text
#默认是8648
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
