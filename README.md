# PulseOps · NapCat QQ 群电话音乐机器人

PulseOps 是一个运行在 Windows 上的 QQ 群电话点歌机器人。项目通过 NapCat/OneBot 接收群消息，通过酷狗音乐搜索并缓存歌曲，再使用桌面自动化加入 QQ 群语音通话，将音乐注入 QQ 麦克风通道。歌曲播放完成后，机器人会自动点击通话窗口中的红色“退出通话”按钮，不会结束其他群成员的通话。

项目同时提供一个企业级可视化管理后台，可查看运行状态、NapCat 连接、通话桥接器、音乐库、酷狗登录状态、点歌记录和实时日志。

## 功能特性

- 使用 `/music 音乐名` 在 QQ 群内点歌。
- 自动判断指定群是否存在语音通话。
- 自动打开正确群聊并加入语音通话。
- 本地音乐优先，未命中时自动搜索酷狗音乐。
- 酷狗 App 扫码登录，会话仅保存在本机。
- 优先请求 320kbps 音源，不可用时回退至 128kbps。
- 将播放器输出路由至 `VoiceMeeter Aux Input`。
- 将 QQ 麦克风路由至 `VoiceMeeter Aux Output`。
- 统一使用 24-bit、48kHz 音频格式并控制增益，减少爆音和二次重采样。
- 歌曲自然播放完成或执行 `/music stop` 后自动退出电话。
- 切换歌曲时不会误退出当前电话。
- 可视化后台、实时日志、音乐库管理和运行状态统计。

## 工作流程

```text
群成员发送 /music 音乐名
          ↓
NapCat WebSocket 推送群消息
          ↓
检测该群是否存在语音通话
          ↓
搜索本地音乐 / 酷狗在线曲库
          ↓
下载并缓存可播放音源
          ↓
自动加入 QQ 群语音通话
          ↓
播放器 → VoiceMeeter Aux Input
          ↓
VoiceMeeter Aux Output → QQ 麦克风
          ↓
播放完成后点击红色“退出通话”
```

## 系统要求

推荐环境：

- Windows 10 或 Windows 11。
- Node.js 18 或更高版本。
- Python 3.10 或更高版本。
- Windows 桌面版 QQ，并保持账号在线。
- NapCat Framework/OneBot 11。
- `VB-Audio VoiceMeeter AUX VAIO` 虚拟音频驱动。

Python 依赖：

```powershell
pip install pywinauto pillow pywin32
```

Node.js 依赖会由 `start.bat` 自动安装，也可以手动执行：

```powershell
cd C:\Users\Administrator\Desktop\qq_musicbot
npm.cmd install
npm.cmd install --omit=dev --prefix services\KuGouMusicApi-main
```

## 端口说明

| 端口 | 服务 | 用途 |
| --- | --- | --- |
| `3000` | NapCat HTTP | 调用 OneBot API、发送群消息 |
| `3001` | NapCat WebSocket | 接收群消息和 OneBot 事件 |
| `3210` | PulseOps 控制台 | 管理后台和机器人主服务 |
| `3211` | QQ Call Bridge | 群电话检测、加入、播放和退出 |
| `3212` | KuGouMusicApi | 酷狗登录、搜索和歌曲 URL 获取 |
| `3213` | 临时音乐房间 | 向群成员提供当前歌曲的临时 Web 播放页面 |
| `6099` | NapCat WebUI | NapCat 自身管理页面，具体以本机配置为准 |

除临时音乐房间默认监听 `0.0.0.0:3213` 外，其余服务默认只监听 `127.0.0.1`。

### 临时音乐房间

成功播放歌曲后，机器人会在群内发送形如 `http://局域网IP:3213/listen/随机令牌` 的临时链接。访问者可以查看当前歌曲、同步播放进度并收听音频。歌曲自然结束、手动停止或被新歌曲替换时，房间与音频地址会自动失效。

如果需要让局域网之外的用户访问，请为 `3213` 端口配置 HTTPS 反向代理，并通过 `SHARE_PUBLIC_BASE_URL` 设置对外地址。不要将管理后台 `3210` 端口一并公开。

## 快速启动

### 1. 启动 QQ 和 NapCat

先登录 Windows 桌面版 QQ，再启动 NapCat。项目内置的 NapCat Framework 可通过以下文件启动：

```text
NapCat.Framework\napiLoader.bat
```

NapCat 必须成功监听：

```text
http://127.0.0.1:3000
ws://127.0.0.1:3001
```

### 2. 检查 OneBot 配置

NapCat OneBot 配置文件位于：

```text
NapCat.Framework\config\onebot11_你的QQ号.json
```

至少启用一个 HTTP 服务端和一个 WebSocket 服务端：

```json
{
  "network": {
    "httpServers": [
      {
        "enable": true,
        "host": "127.0.0.1",
        "port": 3000,
        "token": ""
      }
    ],
    "websocketServers": [
      {
        "enable": true,
        "host": "127.0.0.1",
        "port": 3001,
        "token": ""
      }
    ]
  }
}
```

NapCat 的 OneBot Token 必须与 `config.json` 中的 `napcat.accessToken` 完全一致。这里的 Token 是 OneBot API 访问密码，不是 QQ 登录 Token，也不应向他人泄露。

### 3. 配置机器人

复制示例配置：

```powershell
Copy-Item .\config.example.json .\config.json
```

默认配置：

```json
{
  "server": {
    "host": "127.0.0.1",
    "port": 3210
  },
  "napcat": {
    "httpUrl": "http://127.0.0.1:3000",
    "wsUrl": "ws://127.0.0.1:3001",
    "accessToken": ""
  },
  "bot": {
    "command": "/music",
    "allowedGroups": [],
    "adminUsers": []
  },
  "music": {
    "directory": "./music",
    "extensions": [".mp3", ".flac", ".wav", ".m4a", ".ogg", ".aac"]
  },
  "kugou": {
    "enabled": true,
    "baseUrl": "http://127.0.0.1:3212",
    "port": 3212,
    "serviceDirectory": "./services/KuGouMusicApi-main",
    "sessionFile": "./data/kugou-session.json",
    "cacheDirectory": "./music/kugou-cache"
  },
  "callBridge": {
    "mode": "windows",
    "url": "http://127.0.0.1:3211",
    "token": "",
    "serviceDirectory": "./services/qq-call-bridge"
  }
}
```

`allowedGroups` 为空数组时允许所有群使用。只允许指定群时填写群号：

```json
"allowedGroups": ["417466073", "123456789"]
```

### 4. 启动机器人

双击：

```text
start.bat
```

或在 PowerShell 中运行：

```powershell
cd C:\Users\Administrator\Desktop\qq_musicbot
npm.cmd start
```

正常启动后会看到类似日志：

```text
控制台已启动：http://127.0.0.1:3210
群通话桥接模式：windows
正在连接 NapCat WebSocket：ws://127.0.0.1:3001
```

### 5. 登录酷狗音乐

打开后台：

```text
http://127.0.0.1:3210
```

进入“酷狗音乐”页面：

1. 点击“扫码登录”。
2. 使用酷狗音乐 App 扫描二维码。
3. 在手机上确认授权。
4. 后台显示登录成功后即可在线搜索歌曲。

登录信息保存在：

```text
data\kugou-session.json
```

请勿上传、公开或发送该文件。

## 群指令

| 指令 | 功能 |
| --- | --- |
| `/music 音乐名` | 检测群电话、自动加入并播放音乐 |
| `/music stop` | 停止当前音乐并自动退出电话 |
| `/music status` | 查看当前播放状态 |
| `/music help` | 显示机器人帮助 |

示例：

```text
/music 别怕变老
/music 青花瓷 周杰伦
/music stop
```

如果群内没有电话，机器人会回复：

```text
群聊内没有电话，无法播放音乐。
```

## 音乐搜索与缓存

点歌时的搜索顺序：

1. 搜索 `music` 文件夹中的非酷狗缓存歌曲。
2. 已登录酷狗时，在线搜索当前请求。
3. 优先选择精确歌名和原版歌曲。
4. 自动降低 Live、DJ、Cover、伴奏和片段版本的优先级。
5. 优先请求 320kbps MP3，失败时回退至 128kbps。
6. 下载成功后保存到 `music/kugou-cache`。

本地支持格式：

```text
.mp3 .flac .wav .m4a .ogg .aac
```

添加本地歌曲后，可以在后台“音乐库”页面执行重新扫描，或重启主服务。

## Windows 群电话桥接

`callBridge.mode` 设置为 `windows` 时，主服务会自动启动 `127.0.0.1:3211` 的 Windows 通话桥接器。

桥接器执行以下操作：

- 按群号搜索并打开正确的 QQ 群。
- 通过顶部蓝色“立即加入”按钮检测未加入的群电话。
- 通过“结束通话”状态识别机器人是否已经加入。
- 将 QQ 的麦克风采集绑定到 `VoiceMeeter Aux Output`。
- 将播放器进程绑定到 `VoiceMeeter Aux Input`。
- 打开独立的“语音通话”窗口。
- 歌曲播放完成后识别底部红色“退出通话”按钮并点击。

自动退出只关闭机器人自己的通话窗口。实测退出后群电话仍保持进行，不会挂断其他群成员。

桥接服务接口：

```text
GET  /v1/health
GET  /v1/groups/:groupId/call
POST /v1/groups/:groupId/join
POST /v1/groups/:groupId/play
POST /v1/groups/:groupId/stop
```

健康检查：

```powershell
Invoke-RestMethod http://127.0.0.1:3211/v1/health
```

正常返回示例：

```json
{
  "ok": true,
  "mode": "windows",
  "ocrReady": true,
  "qqAvailable": true,
  "audioRoute": "播放器 → VoiceMeeter Aux Input → VoiceMeeter Aux Output → QQ 麦克风"
}
```

## 音频质量

桥接器默认使用：

- 音源：优先 320kbps MP3。
- VoiceMeeter 输入：24-bit、48kHz、72% 音量。
- VoiceMeeter 输出：24-bit、48kHz、88% 音量。
- 播放器内部音量：85%。

QQ 群电话属于语音通话，会使用语音编码、降噪、回声消除和动态压缩，因此最终音质仍低于本地播放器。为了减少音乐被处理成模糊的人声，建议在 QQ 通话设置中关闭或降低：

- 麦克风降噪。
- 回声消除。
- 自动增益。
- 人声增强。

如果需要接近原始音乐的音质，应使用 QQ 屏幕共享中的“共享电脑声音”，而不是普通群电话麦克风通道。

## 管理后台

后台地址：

```text
http://127.0.0.1:3210
```

主要页面：

- **运行总览**：NapCat、桥接器、酷狗和播放状态。
- **音乐库**：查看本地歌曲与酷狗缓存。
- **酷狗音乐**：扫码登录、搜索歌曲和退出登录。
- **指令与日志**：模拟指令并查看实时日志。
- **系统配置**：查看当前连接地址和运行模式。

后台使用 WebSocket 实时刷新日志和运行状态。

### 管理员 Token 登录

首次启动主服务时会自动生成高强度随机 Token：

```text
data\admin-token.json
```

文件格式：

```json
{
  "token": "自动生成的管理后台Token",
  "createdAt": "生成时间"
}
```

打开后台时，未登录用户会自动跳转到 Three.js 动态登录页。Token 验证成功后，服务器会签发一个有效期为 12 小时的 HttpOnly 会话 Cookie。后台页面、管理 API 和 `/live` 实时 WebSocket 均需要有效会话。

安全机制包括：

- Token 使用加密安全随机数生成。
- 登录 Token 不保存在浏览器 LocalStorage。
- 会话 Cookie 使用 `HttpOnly` 和 `SameSite=Strict`。
- HTTPS 访问时自动添加 `Secure` Cookie 属性。
- 同一来源 15 分钟内连续失败 5 次会触发登录限速。
- 所有页面设置 CSP、禁止 iframe 嵌入并关闭不需要的浏览器权限。
- 服务重启后旧会话自动失效，需要重新登录。

不要把 `data/admin-token.json` 上传到网盘、Git 仓库或发送给其他人。需要更换 Token 时，停止服务、删除该文件并重新启动，系统会自动生成新 Token。

## 公网部署

即使后台已经启用 Token 登录，也不要直接把 3210 端口以明文 HTTP 暴露到公网。推荐让 PulseOps 继续监听 `127.0.0.1:3210`，再通过 Caddy、Nginx、Traefik 或云服务负载均衡器提供 HTTPS。

Caddy 示例：

```caddyfile
musicbot.example.com {
    encode zstd gzip
    reverse_proxy 127.0.0.1:3210
}
```

Nginx 示例：

```nginx
server {
    listen 443 ssl http2;
    server_name musicbot.example.com;

    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3210;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

公网部署建议：

- 仅开放 `80/443`，不要开放 3000、3001、3211、3212 和 6099。
- NapCat、酷狗适配服务和通话桥接器继续监听 `127.0.0.1`。
- 使用受信任的 HTTPS 证书，不要通过公网明文传输 Token。
- 配置云防火墙、安全组或系统防火墙限制来源地址。
- 定期检查 `data/server.out.log` 和后台异常事件。
- 如果只供自己使用，优先使用 VPN、Tailscale、ZeroTier 或固定 IP 白名单。

## 环境变量

除 `config.json` 外，也可以使用环境变量覆盖部分配置。示例见 `.env.example`：

| 环境变量 | 说明 |
| --- | --- |
| `PORT` | 管理后台端口 |
| `NAPCAT_HTTP_URL` | NapCat HTTP 地址 |
| `NAPCAT_WS_URL` | NapCat WebSocket 地址 |
| `NAPCAT_ACCESS_TOKEN` | OneBot 访问 Token |
| `CALL_BRIDGE_MODE` | `windows`、`external`、`mock` 或 `disabled` |
| `CALL_BRIDGE_URL` | 外部或 Windows 桥接器地址 |
| `CALL_BRIDGE_TOKEN` | 外部桥接器鉴权 Token |
| `MUSIC_DIRECTORY` | 本地音乐目录 |
| `SHARE_ENABLED` | 是否启用临时音乐房间 |
| `SHARE_HOST` | 音乐房间监听地址，默认 `0.0.0.0` |
| `SHARE_PORT` | 音乐房间端口，默认 `3213` |
| `SHARE_PUBLIC_BASE_URL` | 机器人发送的公网或反向代理基础地址 |

PowerShell 示例：

```powershell
$env:NAPCAT_ACCESS_TOKEN = "你的OneBotToken"
$env:CALL_BRIDGE_MODE = "windows"
npm.cmd start
```

## 故障排查

### NapCat WebSocket `ECONNREFUSED 127.0.0.1:3001`

表示 3001 端口没有 NapCat WebSocket 服务监听。

检查：

```powershell
Get-NetTCPConnection -LocalPort 3001 -State Listen
```

处理方法：

1. 确认 QQ 和 NapCat 已启动。
2. 确认 OneBot WebSocket 服务端已启用。
3. 确认端口是 `3001`。
4. 修改配置后重启 NapCat。

### 群内有电话但机器人提示没有电话

检查：

- QQ 主窗口是否仍然打开。
- 当前 QQ 是否为新版 Windows 桌面版。
- 群号是否在 `allowedGroups` 中。
- 桥接器健康检查中的 `qqAvailable` 是否为 `true`。
- QQ 界面缩放和 Windows 显示缩放是否发生大幅变化。

视觉检测会识别顶部蓝色“立即加入”按钮和独立通话窗口，不依赖 NapCat 提供群电话 API，因为 OneBot/NapCat 本身没有群电话状态与音频注入接口。

### 能加入电话但别人听不到音乐

检查音频设备是否存在：

```powershell
services\qq-call-bridge\tools\soundvolumeview\SoundVolumeView.exe /scomma services\qq-call-bridge\temp\audio.csv
Select-String services\qq-call-bridge\temp\audio.csv -Pattern "VoiceMeeter"
```

必须同时存在：

```text
VoiceMeeter Aux Input
VoiceMeeter Aux Output
```

播放时应满足：

- `powershell.exe` 的活动 Render 会话位于 VoiceMeeter AUX 设备。
- `QQ.exe` 的活动 Capture 会话位于 VoiceMeeter AUX 设备。

如果 QQ 已经在使用其他麦克风，先退出电话再重新发送点歌指令，让桥接器在加入前完成设备绑定。

### 音质模糊、声音小或爆音

- 确认新下载文件名包含 `320k`。
- 删除旧的 128kbps 缓存后重新点歌。
- 关闭 QQ 麦克风降噪和自动增益。
- 不要手动把 VoiceMeeter 两端同时调到 100%。
- 确认输入和输出都是 48kHz。

### 播放结束后没有自动退出

检查是否存在独立窗口：

```text
窗口标题：语音通话
```

桥接器会在音乐进程退出后识别该窗口底部的红色“退出通话”按钮。如果 QQ 大版本更新导致按钮颜色或布局变化，需要同步调整 `services/qq-call-bridge/qq_ui.py`。

### 酷狗无法获取歌曲

可能原因：

- 二维码登录过期。
- 歌曲需要 VIP 权限。
- 地区或版权限制。
- 酷狗接口暂时不可用。
- 当前账号没有该歌曲的播放权限。

可以在后台退出酷狗后重新扫码登录。

## 开发与检查

开发模式：

```powershell
npm.cmd run dev
```

语法检查：

```powershell
npm.cmd run check
```

Python 语法检查：

```powershell
python -m py_compile services\qq-call-bridge\qq_ui.py
```

## 目录结构

```text
qq_musicbot/
├─ src/                          机器人、NapCat 客户端和后端服务
├─ public/                       企业级可视化管理后台
├─ music/                        本地音乐目录
│  └─ kugou-cache/               酷狗歌曲缓存
├─ services/
│  ├─ KuGouMusicApi-main/        酷狗本地适配服务
│  └─ qq-call-bridge/            Windows QQ 通话桥接器
├─ data/                         会话、运行状态和日志
├─ NapCat.Framework/             NapCat Framework
├─ config.json                   当前配置
├─ config.example.json           配置示例
├─ start.bat                     Windows 快速启动脚本
└─ README.md                     项目文档
```

## 安全与隐私

- 不要公开 NapCat WebUI Token、OneBot Token 或 QQ 登录信息。
- 不要公开 `data/kugou-session.json`。
- 使用 `allowedGroups` 限制机器人可工作的群。
- 管理后台默认仅监听 `127.0.0.1`；公网访问必须使用 HTTPS 反向代理。
- 不要从不可信来源下载或运行虚拟声卡、驱动和可执行文件。
- `services/qq-call-bridge/tools/soundvolumeview` 仅用于本机音频设备路由。
- 机器人只能播放当前酷狗账号有权访问的内容。

## 版权声明

本项目仅用于个人学习、自动化研究和合法场景。使用者应遵守 QQ、NapCat、酷狗音乐及相关服务的使用条款，并自行确保拥有音乐播放、传播和使用权限。

酷狗适配层来源于 `MakcRe/KuGouMusicApi`，其源码和许可证保留在 `services/KuGouMusicApi-main`。
