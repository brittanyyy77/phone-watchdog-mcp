# phone-watchdog-mcp

AI 远程监控 iPhone 状态并控制 App 锁定。

VPS 上跑一个轻量服务，手机快捷指令定期上报状态和截屏，AI 通过 MCP 读取数据并下发指令。

## 架构

```
AI 客户端 (Aru / Claude)
    ↓ MCP (streamable-http)
https://你的域名/mcp
    ↓ Cloudflare Tunnel
VPS:18770 (Python 服务)
    ↑ 上传截屏 / App事件 / 屏幕时间 / ACK
    ↓ 下发命令 (lock/unlock/screenshot)
iPhone 快捷指令
    ↓ 执行 Jomo Session
    ↓ Bark 推送通知
```

## v2 新增功能

| 功能 | 说明 |
|------|------|
| 命令 ACK | 手机执行完命令后回报确认，AI 能确认是否真的执行了 |
| 命令去重 | 每条命令带唯一 ID，手机不会重复执行 |
| 手机心跳 | 每次轮询就是一次心跳，超过 90 秒没心跳就知道断了 |
| watchdog_health | 一个接口看全部状态：心跳、命令、ACK、数据新鲜度 |

## 快速开始

### 1. 服务端（VPS）

```bash
git clone https://github.com/brittanyyy77/phone-watchdog-mcp.git
cd phone-watchdog-mcp

cp .env.example .env
chmod 600 .env
nano .env  # 填入 Token 和 Bark URL

pip3 install -r requirements.txt
python3 server.py
```

### 2. Cloudflare Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create watchdog
cloudflared tunnel route dns watchdog watchdog.你的域名.com
```

`~/.cloudflared/config.yml`：

```yaml
tunnel: 你的Tunnel_ID
credentials-file: ~/.cloudflared/你的Tunnel_ID.json

ingress:
  - hostname: watchdog.你的域名.com
    service: http://localhost:18770
  - service: http_status:404
```

```bash
cloudflared tunnel run watchdog
```

### 3. 手机端（iOS 快捷指令）

所有请求加 Header：`Authorization: Bearer 你的TOKEN`

**指令 1：命令轮询（每 30 秒跑一次）**

1. GET `https://你的域名/command`
2. 获取返回的 `command` 和 `id` 字段
3. 如果 `command` 等于 `none` → 什么都不做
4. 如果 `command` 等于 `lock` → 运行 Jomo「Start Session」
5. 如果 `command` 等于 `unlock` → 运行 Jomo「Stop Session」
6. 如果 `command` 等于 `screenshot` → 截取屏幕截图 → POST 到 `/upload`
7. 执行完成后 POST `/ack`：
```json
{"command_id": "返回的id", "status": "done"}
```

> **关键**：每次拿到命令先记下 `id`，执行完后发 ACK。如果这次拿到的 `id` 跟上次一样，说明是重复投递，跳过不执行。

**指令 2：App 事件上报（自动化触发）**

POST `https://你的域名/event`
```json
{"app": "App名称", "event": "open"}
```

**指令 3：屏幕时间上报（每天一次）**

1. Jomo 动作：Get Today's Screen Time
2. POST `https://你的域名/screen-time`

### 4. 接入 AI 客户端

| 配置项 | 填写内容 |
|--------|----------|
| 名称 | 手机监控 |
| URL | `https://watchdog.你的域名.com/mcp` |
| 传输类型 | streamable-http |
| 鉴权方式 | configured_headers |
| Header | Authorization: Bearer 你的TOKEN |

## MCP 工具列表

| 工具 | 说明 |
|------|------|
| `get_screenshot` | 读取最新截图 |
| `request_screenshot` | 下发截屏指令（返回 command_id） |
| `get_phone_status` | 当前 App、心跳状态、截图时间 |
| `get_app_history` | 最近 App 开关记录 |
| `get_screen_time` | 今日屏幕使用时长 |
| `lock_apps` | 锁手机（返回 command_id，同时 Bark 推送） |
| `unlock_apps` | 解锁手机（返回 command_id） |
| `send_notification` | Bark 推送通知 |
| `watchdog_health` | 全局健康检查：心跳、命令、ACK、数据新鲜度 |

## watchdog_health 返回示例

```json
{
  "server": {"ok": true, "time": "2026-11-17 12:00:00"},
  "phone": {
    "alive": true,
    "last_seen": "2026-11-17 11:59:42",
    "silence_seconds": 18,
    "timeout_threshold": 90
  },
  "screenshot": {"exists": true, "taken_at": "2026-11-17 11:55:00"},
  "current_app": "WeChat",
  "pending_command": {
    "command": "lock",
    "id": "a1b2c3d4e5f6",
    "created_at": "2026-11-17 11:58:00",
    "delivered": true,
    "age_seconds": 120,
    "stuck": false
  },
  "last_ack": {
    "command_id": "a1b2c3d4e5f6",
    "status": "done",
    "acked_at": "2026-11-17 11:58:35"
  },
  "screen_time_updated_at": "2026-11-17 08:00:00"
}
```

看这一个接口就知道：
- 手机是不是活的（心跳）
- 命令发了没有（pending_command）
- 手机执行了没有（last_ack）
- 命令是不是卡住了（stuck）
- 截图多久了（screenshot.taken_at）

## 踩坑记录

| # | 现象 | 原因 | 解法 |
|---|------|------|------|
| 1 | 截图命令发了但收不到图 | 快捷指令轮询间隔太长 | 轮询设 30 秒，用 watchdog_health 检查心跳 |
| 2 | lock 发了但手机没锁 | Jomo Session 没配置或快捷指令没跑 | 检查 watchdog_health 的 ACK 状态 |
| 3 | 命令被重复执行 | 网络拖动手机拿了两次 | v2 每条命令有唯一 ID，快捷指令里记住上次的 ID，相同就跳过 |
| 4 | watchdog_health 显示 stuck | 命令超时没被执行 | 手机可能没在轮询，检查快捷指令自动化 |
| 5 | 心跳断了但手机没事 | iOS 后台限制快捷指令执行 | 正常现象，iOS 会限制后台自动化频率 |

## 安全说明

- Token 至少 32 字节：`openssl rand -hex 32`
- 服务绑定 127.0.0.1，不裸露端口
- 截图和 App 记录是敏感数据，不要提交到公开仓库
- .env 和 data/ 已在 .gitignore 里

## 致谢

- [Jomo](https://www.jomo.so) — iOS App 锁定
- [Bark](https://github.com/Finb/Bark) — iOS 推送通知

## License

MIT
