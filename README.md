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
    ↑ 上传截屏 / App事件 / 屏幕时间
    ↓ 下发命令 (lock/unlock/screenshot)
iPhone 快捷指令
    ↓ 执行 Jomo Session
    ↓ Bark 推送通知
```

## 功能

| 功能 | 说明 |
|------|------|
| 远程截屏 | AI 下发截屏指令，手机自动截图上传，AI 直接查看 |
| App 监控 | 快捷指令自动上报 App 开关事件，AI 知道你在用什么 |
| 屏幕时间 | 从 Jomo 获取今日屏幕使用时长 |
| 锁 App | AI 下发 lock 指令，Jomo 启动锁定 Session |
| 解锁 App | AI 下发 unlock 指令，Jomo 停止 Session |
| Bark 推送 | AI 给你手机发通知，锁的时候告诉你是谁干的 |

## 快速开始

### 1. 服务端（VPS）

```bash
git clone https://github.com/brittanyyy77/phone-watchdog-mcp.git
cd phone-watchdog-mcp

cp .env.example .env
chmod 600 .env
nano .env  # 填入你的 Token 和 Bark URL

pip3 install flask requests pillow
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

需要三个快捷指令，全部加 Header：`Authorization: Bearer 你的TOKEN`

**指令 1：命令轮询（核心，每 30 秒跑一次）**

1. GET `https://你的域名/command`
2. 获取返回的 `command` 字段
3. 如果等于 `lock` → 运行 Jomo 「Start Session」
4. 如果等于 `unlock` → 运行 Jomo 「Stop Session」
5. 如果等于 `screenshot` → 截取屏幕截图 → POST 到 `https://你的域名/upload`

**指令 2：App 事件上报（自动化触发）**

触发条件：某个 App 打开/关闭

POST `https://你的域名/event`
```json
{"app": "App名称", "event": "open"}
```

**指令 3：屏幕时间上报（每天一次）**

1. Jomo 动作：Get Today's Screen Time
2. POST `https://你的域名/screen-time`，请求体为 Jomo 输出

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
| `request_screenshot` | 下发截屏指令，手机轮询到后自动截图上传 |
| `get_phone_status` | 当前 App、最后事件、最新截图时间 |
| `get_app_history` | 最近 App 开关记录 |
| `get_screen_time` | 今日屏幕使用时长 |
| `lock_apps` | 下发锁定指令，Jomo 启动 Session |
| `unlock_apps` | 下发解锁指令，Jomo 停止 Session |
| `send_notification` | 通过 Bark 给手机推送通知 |

## 踩坑记录

| # | 现象 | 原因 | 解法 |
|---|------|------|------|
| 1 | 截图命令发了但收不到图 | 快捷指令轮询间隔太长 | 轮询间隔设为 30 秒或更短 |
| 2 | lock 命令下发了但手机没锁 | 快捷指令没跑或 Jomo Session 没配置 | 确认快捷指令自动化开着，Jomo 里有预配的 Session |
| 3 | 截图只看到黑屏 | iOS 锁屏时截图就是黑的 | 正常现象，说明手机在休眠 |
| 4 | Bark 推送收不到 | Bark URL 填错或 Bark App 没装 | 确认 .env 里 BARK_URL 格式正确 |
| 5 | 命令被重复执行 | 快捷指令轮询太快，同一个命令拿了两次 | 服务端命令是一次性的，读取后自动删除 |

## 安全说明

- Token 至少 32 字节：`openssl rand -hex 32`
- 服务绑定 127.0.0.1，不裸露端口
- 截图和 App 记录是敏感数据，不要提交到公开仓库
- .env 和 data/ 目录已在 .gitignore 里

## 致谢

- [Jomo](https://www.jomo.so) — iOS 屏幕时间管理与 App 锁定
- [Bark](https://github.com/Finb/Bark) — iOS 推送通知

## License

MIT
