# 无限画布 + Muse2API + WorkBuddy 接入教程

> 从零部署一套「本地 AI 影像工作台」：浏览器里的无限画布负责编排，Muse2API 提供生图/生视频能力，WorkBuddy 通过 MCP 直接操作画布。
> 本文所有命令与路径均按 Windows + 本机实测整理，非推测。

---

## 0. 这套东西是什么

```
┌─────────────────────────────────────────────────────────────┐
│  浏览器：无限画布（localhost:3000）                          │
│  ├─ 画布项目 / 节点 / 连线 / 素材   → 浏览器 IndexedDB 本地存储 │
│  └─ 生成请求 → OpenAI 兼容接口                                │
└───────────────┬─────────────────────────┬───────────────────┘
                │ 生成请求                 │ SSE（侧边栏 Agent）
                ▼                         ▼
   ┌────────────────────────┐   ┌──────────────────────────┐
   │ Muse2API (127.0.0.1:18610) │   │ Canvas Agent (17371)  │
   │ 逆向 muse.ai 网页能力      │   │ 用 Codex 驱动画布       │
   │ 输出标准 OpenAI 接口       │   └──────────────────────────┘
   └───────────┬────────────┘                 ▲
               │ 无头浏览器 / CDP              │ MCP（stdio）
               ▼                              │
        ┌─────────────┐              ┌────────┴─────────┐
        │  muse.ai    │              │    WorkBuddy     │
        └─────────────┘              └──────────────────┘
```

三个服务，各自独立，**缺一个就有功能不可用**：

| 服务 | 端口 | 缺了会怎样 |
|---|---|---|
| 无限画布前端 | 3000 | 打不开页面 |
| Muse2API 网关 | 18610 | 画布能开，但生图/生视频全失败 |
| Canvas Agent | 17371 | 画布能开，但 Agent 面板和 MCP 工具不可用 |

---

## 1. 环境准备

### 必装

| 软件 | 版本要求 | 检查命令 |
|---|---|---|
| Node.js | **20.19+ 或 22.12+**（vite 7 硬要求） | `node -v` |
| Python | 3.10+（Muse2API 要求，3.12 实测可用） | `python -V` |
| Git | 任意较新版本 | `git --version` |
| Chrome | 最新版（Muse2API 靠它跑无头会话） | — |

### 可选

- **ffmpeg**：只在你要剪辑、拼片、转码时需要。建议放 `C:\ffmpeg\bin\`，并确认 `ffmpeg -version` 能跑。
- **注册一个 muse.ai 账号**：Muse2API 是把网页端能力转成 API，所以必须有登录态。

### 国内网络注意

npm 和 pip 都要换源，否则大概率卡死：

```bash
npm config set registry https://registry.npmmirror.com
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
```

---

## 2. 部署 Muse2API 网关

仓库：<https://github.com/czg86389-hub/muse2api>

### 2.1 拉代码 + 装依赖

```bash
D:
mkdir muse-canvas && cd muse-canvas
git clone https://github.com/czg86389-hub/muse2api.git
cd muse2api

python -m venv venv
venv\Scripts\python.exe -m pip install -U pip
venv\Scripts\python.exe -m pip install -r requirements.txt
```

> 如果 `git clone` 走不通，用 `https://gh-proxy.com/https://github.com/...` 或直接下 zip。

### 2.2 配置 `.env`

复制 `.env.example` 为 `.env`（没有示例就新建），填这几项：

```ini
MUSE2API_KEY=m2a_换成你自己的随机串      # 画布里要填的 API Key
MUSE2API_HOST=127.0.0.1
MUSE2API_PORT=18610
MUSE2API_PUBLIC_BASE=http://localhost:18610   # 返回的媒体地址前缀
MUSE2API_CONCURRENCY=2                        # 同时跑几个生成任务
MUSE2API_REF_IMAGE_LOCAL_FILES=0              # 是否允许参考图读本地文件
MUSE2API_REF_IMAGE_PRIVATE_FETCH=0            # 是否允许参考图抓本机地址
```

`MUSE2API_KEY` 就当作密码，别提交到公开仓库。

### 2.3 首启动

```bash
venv\Scripts\python.exe -m uvicorn app:app --host 127.0.0.1 --port 18610
```

看到 `Uvicorn running on http://127.0.0.1:18610` 即成功。另开一个终端验证：

```bash
curl http://127.0.0.1:18610/healthz
# {"status":"ok", ...}
```

### 2.4 导入 muse.ai 登录态（关键一步）

网关本身不带账号，必须把你的 muse.ai 会话灌进去。项目自带抓取助手：

```bash
venv\Scripts\python.exe tools\get_muse_cookie.py --base http://127.0.0.1:18610 --key m2a_你的key
```

它会弹一个**独立 Chrome 窗口**（不影响你日常浏览器），你在这个窗口里正常登录 muse.ai。登录成功后脚本自动抓取 `hatch_sess` 等 cookie 并回填，最后打印账号 ID 和有效期。

验证：

```bash
curl -H "Authorization: Bearer m2a_你的key" http://127.0.0.1:18610/admin/accounts
```

看到 `"ok": true` 就通了。也可以直接开浏览器看管理页：<http://127.0.0.1:18610/admin>

### 2.5 出图自测

```bash
curl -X POST http://127.0.0.1:18610/v1/images/generations ^
  -H "Authorization: Bearer m2a_你的key" -H "Content-Type: application/json" ^
  -d "{\"model\":\"muse-image\",\"prompt\":\"a red apple on a white table\",\"size\":\"1280x720\",\"aspect_ratio\":\"16:9\"}"
```

返回里带 `url` 字段就说明整条链路通了。

---

## 3. 部署无限画布

仓库：<https://github.com/basketikun/infinite-canvas>（MIT，当前 v0.20.0）

### 3.1 拉代码 + 构建

```bash
cd D:\muse-canvas
git clone https://github.com/basketikun/infinite-canvas.git
cd infinite-canvas\web

npm install --registry=https://registry.npmmirror.com --no-audit --no-fund
npm run build
```

### 3.2 启动

```bash
npm start
# 实际执行：vite preview --host 0.0.0.0 --port 3000
```

浏览器打开 <http://localhost:3000>。

> **不要用 `file://` 直接打开 `dist/index.html`**，那样页面空白，必须走 http 服务。

### 3.3 关于数据存储（务必先读）

画布的**项目、节点、素材、API Key 全部存在浏览器本地**（IndexedDB），不上传服务器。由此有三个后果：

1. **`localhost:3000` 和 `127.0.0.1:3000` 是两个不同的源**，存储互不相通。今天用哪个地址建的画布，以后就得用同一个地址打开。
2. 换浏览器 / 换 Chrome 配置文件（user-data-dir）也看不到之前的画布。
3. Chrome 的「清除浏览数据」如果勾了「Cookie 及其他网站数据」，画布会一起没。

**建议固定一个专用浏览器配置文件**，专门开画布：

```bash
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --user-data-dir="D:\muse-canvas\chrome-profile" ^
  --new-window "http://localhost:3000/"
```

以后就用这个 profile 开画布，日常浏览器随便清缓存都不影响。

---

## 4. 在画布中配置模型

进入画布 → **设置 / 配置**（`/config`），新增一个「渠道」：

| 字段 | 值 |
|---|---|
| Base URL | `http://127.0.0.1:18610/v1` ← **必须带 `/v1`** |
| API Key | `m2a_你的key` |
| 生图模型 | `muse-image` |
| 生视频模型 | `muse-video` |
| 生文模型 | `muse-chat`（可选，走同一网关） |

然后到「生成偏好」里设置默认值：

- 默认生图分辨率：`2048x1152`（16:9）或 `1152x2048`（竖屏）
- 默认视频分辨率：`1080P`
- **已有内容的节点再次生成时**：建议选 **「原地重新生成（覆盖当前节点）」**，否则新结果会固定落在源节点右侧 96px 处，容易跑到视口外，看起来像"没生成"。

### 4.1 手动验证

在画布上新建一个图片节点，输入 `a red apple`，点生成。30~60 秒出图即配置正确。

---

## 5. 接入 WorkBuddy（MCP）

到这里画布已经能用，但只能手动点。接入 MCP 后，可以直接在 WorkBuddy 对话里说「读一下当前画布」「在这里加个节点」。

### 5.1 拿到 canvas-agent

```bash
npx -y @basketikun/canvas-agent --version
```

这条命令会把它下载到 npx 缓存（约 436MB，内含捆绑的 Codex）。记下缓存目录，形如：

```
D:\env\WorkBuddyenv\cache\npm\_npx\<hash>\node_modules\@basketikun\canvas-agent
```

### 5.2 建一个固定目录（避免缓存被清）

把包复制到一个稳定位置，并给依赖建 **目录联接**（不要复制 436MB）：

```powershell
# 1. 复制包本体
New-Item -ItemType Directory -Force "D:\muse-canvas\canvas-agent-mcp"
Copy-Item "D:\env\WorkBuddyenv\cache\npm\_npx\<hash>\node_modules\@basketikun\canvas-agent\*" `
  "D:\muse-canvas\canvas-agent-mcp\" -Recurse -Force

# 2. 依赖用 junction 指过去
New-Item -ItemType Junction -Path "D:\muse-canvas\canvas-agent-mcp\node_modules" `
  -Target "D:\env\WorkBuddyenv\cache\npm\_npx\<hash>\node_modules"
```

### 5.3 验证 MCP 能起来

```powershell
& "C:\Users\<你>\.workbuddy\binaries\node\versions\22.22.2-6\node.exe" `
  "D:\muse-canvas\canvas-agent-mcp\dist\index.js" mcp
```

正常情况下它会**静默等待 stdin**（MCP 走 stdio），不报错就是好的。若报 `Cannot find package 'express'`，说明 junction 没建对。

### 5.4 写进 WorkBuddy 配置

编辑 `C:\Users\<你>\.workbuddy\mcp.json`，在 `mcpServers` 里加：

```json
{
  "mcpServers": {
    "infinite-canvas": {
      "command": "C:\\Users\\<你>\\.workbuddy\\binaries\\node\\versions\\22.22.2-6\\node.exe",
      "args": [
        "D:\\muse-canvas\\canvas-agent-mcp\\dist\\index.js",
        "mcp"
      ]
    }
  }
}
```

注意三点：

- `command` 用**绝对路径**，别依赖 PATH
- 路径用**反斜杠转义**（`\\`）
- 不要用 `.bat` / `.cmd` 做包装，管道下容易卡住

### 5.5 启用

1. **完整退出 WorkBuddy 再打开**（MCP 只在启动时读配置）
2. 进「连接器管理」页，找到 `infinite-canvas`，点 **信任**
3. 确认 **Canvas Agent 服务在跑**（见 6.1）+ **浏览器画布页面开着**

第 3 条最容易忽略：MCP 只是客户端，真正动手的是浏览器里的页面。

### 5.6 能用的工具（34 个）

```
读取   canvas_get_state / canvas_get_selection / canvas_list_projects / canvas_export_snapshot
建改   canvas_create_node / canvas_create_text_node / canvas_update_node / canvas_move_nodes
       canvas_resize_node / canvas_delete_nodes / canvas_apply_ops
连接   canvas_connect_nodes / canvas_select_nodes / canvas_set_viewport
生成   canvas_generate_image / canvas_generate_video / canvas_generate_audio / canvas_generate_text
       canvas_run_generation / generation_get_status
工作台 workbench_image_* / workbench_video_*
其他   site_navigate / canvas_create_generation_flow / canvas_create_config_node
```

对话示例：

- 「读一下当前画布，列出所有节点」
- 「在我选中的节点右边 96px 加一个文本节点，内容是今天的拍摄清单」
- 「把这三张图排成一行，间距 40」
- 「用这个节点生一张 16:9 的图，提示词：清晨的厨房」

---

## 6. 一键启动（三个脚本）

三个服务都要常驻，建议各做一个 bat 双击启动。

### 6.1 Canvas Agent

`D:\muse-canvas\canvas-agent-mcp\start-agent.bat`

```bat
@echo off
title Infinite Canvas Agent
cd /d "D:\muse-canvas\canvas-agent-mcp"
"C:\Users\<你>\.workbuddy\binaries\node\versions\22.22.2-6\node.exe" dist\index.js
pause
```

启动后窗口会打印：

```
Local URL: http://127.0.0.1:17371
Connect token: xxxxxxxx
```

这串 token 填到画布的 Agent 设置里。

### 6.2 画布前端

`D:\muse-canvas\infinite-canvas\start-canvas.bat`

```bat
@echo off
title Infinite Canvas
cd /d "%~dp0web"
start "" http://localhost:3000
call npm start
pause
```

### 6.3 Muse 网关

`D:\muse-canvas\muse2api\start-muse.bat`

```bat
@echo off
title Muse2API Gateway
cd /d "%~dp0"
set HTTP_PROXY=http://127.0.0.1:7890
set HTTPS_PROXY=http://127.0.0.1:7890
set NO_PROXY=localhost,127.0.0.1
start "" http://127.0.0.1:18610/admin
.\venv\Scripts\python.exe -m uvicorn app:app --host 127.0.0.1 --port 18610
pause
```

> 有代理才需要那三行 `set`；没有就删掉。`NO_PROXY` 必须包含本机，否则自己请求自己会被代理坑。

**三个窗口都别关**，关掉对应功能就停。

---

## 7. 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 画布打开是空的，项目全没了 | 换了访问地址（`localhost` ↔ `127.0.0.1`）或换了浏览器配置 | 用**建画布时的那个地址**和那个 Chrome profile 打开；先确认 `chrome-profile` 目录里有没有 `http_localhost_3000.indexeddb.leveldb` |
| `localhost 拒绝了连接` | 服务没起 / 端口被占 / 清过缓存 | `netstat -ano \| findstr :3000` 看有没有 LISTENING；看黑窗口有没有报错 |
| 画布能开，生成全失败 | 网关没起 | `curl http://127.0.0.1:18610/healthz` 必须返回 ok |
| 生成报 `401` 或"会话失效" | muse.ai cookie 过期 | 重跑 `tools\get_muse_cookie.py` 重新导入 |
| 生成报 `400`，日志有 `不是图片（application/octet-stream）` | 网关媒体接口没设 MIME，`.webp` 被当二进制，导致「用生成图当参考图」失败 | 在 `app.py` 的 `get_media` 里按扩展名指定 `media_type`（`.webp→image/webp`、`.mp4→video/mp4`） |
| 参考图报「仅允许公网地址」 | 默认禁止抓本机/内网地址 | `.env` 设 `MUSE2API_REF_IMAGE_PRIVATE_FETCH=1` 后重启 |
| 生成一直转圈不出结果 | 画布轮询上限 120 次 × 2.5 秒 = **300 秒**，慢任务会超时 | 一次只提一条；或调小并发、错峰跑 |
| 视频提交就失败 | 画幅不是 16:9 | Muse 只支持 16:9 横屏，竖屏要后期裁 |
| 视频只有 1~2 秒 | 静态人像镜头撑不住长时长，模型会自己截短 | 见 8.2，改用有明确动作的镜头，或接受慢放补时长 |
| 在线版画布（https）请求本地网关被浏览器拦 | Chrome 私有网络访问（PNA）限制 | 用本地 `localhost:3000` 而不是在线版；或在网关侧补 `Access-Control-Allow-Private-Network: true` |
| 侧边栏 Codex 报错 `chatgpt.com` 连不上 | 代理没覆盖 Codex 子进程 | 配好系统代理；或干脆不用 Codex，改用 WorkBuddy 的 MCP |

---

## 8. 提示词规范（决定成败的关键）

Muse 不是 Sora/可灵，有一批硬限制。完整规范见 `muse2api/GPT_PROMPT_SPEC.md`，这里列最关键的。

### 8.1 视频

- **只能 16:9**，9:16 一律失败
- 时长按 **10 秒** 提交最稳
- 提示词 **≤300 字**，只写三件事：**镜头怎么动 + 主体做什么 + 环境是什么**
- **一个镜头只写一个主动作**
- 有首尾帧就传两张，这是控构图最有效的手段

### 8.2 静态人像的真实限制

实测：一张人像 + 固定机位的提示词，Muse 要 10 秒会自己崩（试三版都一样：头歪、笑容变大、镜头乱推、背景出现路人），最终只给 **2 秒**。

对策按优先级：

1. 提示词里给一个**明确动作**（走近、转身、抬手递物、翻页）
2. 接受 2 秒片段，后期 `minterpolate` 补帧 + 慢放拉长
3. 换其它视频模型

### 8.3 禁用词 → 替换

| 不要写 | 改成 |
|---|---|
| 睡着、躺卧、呼吸起伏 | 安静闭眼休息 |
| 奔跑、孩子跑过背景 | 缓慢走过、全程脚踩地面 |
| 拿刀切开、血水、内脏 | 处理食材、在砧板上整理 |
| 大哭、泪流满面 | 眼眶泛红、抿嘴、无声落泪 |
| 任何知名 IP 角色 | 原创角色描述 |

出现这些，模型会**返回一段文字而不是视频**，那不是成功。

### 8.4 人物一致性

只有一条路：**先生成一张角色定妆图并落到画布，之后每镜都引用它作为参考图**。靠文字描述"同一个女孩"是保不住的，尤其多人同框。

---

## 9. 目录结构速查

```
D:\muse-canvas\
├─ muse2api\                     Muse2API 网关
│  ├─ app.py                     主服务（含 get_media 的 MIME 修复）
│  ├─ .env                       KEY / 端口 / 代理
│  ├─ data\accounts.json         账号池
│  ├─ data\media\                生成产物
│  ├─ tools\get_muse_cookie.py   cookie 导入助手
│  ├─ GPT_PROMPT_SPEC.md         给 GPT 的提示词规范
│  └─ start-muse.bat
├─ infinite-canvas\              无限画布
│  ├─ web\                       前端（vite）
│  ├─ canvas-agent\              agent 源码
│  └─ start-canvas.bat
├─ canvas-agent-mcp\             MCP 运行目录
│  ├─ dist\                      canvas-agent 构建产物
│  ├─ node_modules\              → junction 到 npx 缓存
│  └─ start-agent.bat
├─ chrome-profile\               专用 Chrome 配置（画布数据在这里）
└─ open-canvas.bat               用该 profile 打开画布
```

---

## 10. 上线前自检

按顺序跑一遍，全过就算部署成功：

```bash
# 1 三个服务
curl http://127.0.0.1:18610/healthz          # {"status":"ok"}
curl -o /dev/null -w "%{http_code}" http://localhost:3000/   # 200
curl -o /dev/null -w "%{http_code}" http://127.0.0.1:17371/  # 401（有响应即活）

# 2 账号可用
curl -H "Authorization: Bearer m2a_你的key" http://127.0.0.1:18610/admin/accounts
# 看 "ok": true

# 3 出图
curl -X POST http://127.0.0.1:18610/v1/images/generations ^
  -H "Authorization: Bearer m2a_你的key" -H "Content-Type: application/json" ^
  -d "{\"model\":\"muse-image\",\"prompt\":\"a red apple\",\"size\":\"1280x720\",\"aspect_ratio\":\"16:9\"}"
# 返回带 url 即通

# 4 画布有数据
# 浏览器打开 http://localhost:3000/canvas，能看到项目列表

# 5 MCP 生效
# WorkBuddy 里说「读一下当前画布」，能返回节点即通
```

---

## 11. 已知边界（别指望它做这些）

- **竖屏视频**：Muse 只出 16:9，竖屏只能裁，会损失构图
- **可读文字**：画面里的文字基本是乱码，别让它写字幕
- **多人一致性**：人物越多越容易崩，先做全员定妆图
- **长镜头**：单条 10 秒封顶，静态镜头还会缩短
- **并发**：默认 2，调高容易排队超时
- **数据同步**：画布数据只在本地浏览器，没有服务端，换机器不跟随

---

_本文基于 2026-11 实测环境整理：Win11 + Node 22.22.2 + Python 3.12 + infinite-canvas v0.20.0 + canvas-agent 0.6.0 + muse2api（FastAPI）。_
