# bing2api

[简体中文](README.zh-CN.md) | [English](README.md)

<div align="center">

**面向 Bing Video Creator 的 OpenAI 兼容视频 API 服务**

</div>

---

## ✨ 功能特性

### 核心能力
- 🎬 **文生视频** - 文本生成视频
- 🖼️ **图生视频** - 上传图片生成视频
- 🧭 **快/慢速模式** - 显式模型 ID 控制
- 🔄 **异步任务** - 稳定的轮询与结果返回流程
- 🎯 **OpenAI 兼容** - `/v1/videos/generations` 与 `/v1/models`

### 生产特性
- 👥 **账户池路由** - 多账户并发与自动切换
- 🧪 **配额检测** - 快速模式配额识别
- 📦 **SQLite 持久化** - 账户/任务存储
- 🧰 **管理后台** - 账号导入与会话维护
- ⚙️ **运行时设置** - 不重启即可更新 Key 与代理

---

## 🚀 快速开始

### 前置要求
- Python 3.8+
- 可用的 Bing 账户 Cookie

### 本地运行
```bash
git clone git@github.com:jiwgxo/bing2api.git
cd bing2api
pip install -r requirements.txt
PYTHONPATH=src python -m bing_api.api.app
```

启动后访问管理页：`http://localhost:8000/manage`

默认后台账号：
- 用户名：`admin`
- 密码：`admin123`

### Docker（推荐）
```bash
git clone git@github.com:jiwgxo/bing2api.git
cd bing2api
docker compose up -d --build
```

启动后访问管理页：`http://localhost:8000/manage`

默认后台账号：
- 用户名：`admin`
- 密码：`admin123`

---

## 🔌 OpenAI 兼容接口

### 列出模型
```bash
curl http://localhost:8000/v1/models \
  -H "Authorization: Bearer bing-demo-key"
```

### 生成视频（竖屏快）
```bash
curl http://localhost:8000/v1/videos/generations \
  -H "Authorization: Bearer bing-demo-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sora-v2-portrait-fast",
    "prompt": "a cat",
    "async": true
  }'
```

### 生成视频（横屏慢）
```bash
curl http://localhost:8000/v1/videos/generations \
  -H "Authorization: Bearer bing-demo-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sora-v2-landscape-slow",
    "prompt": "a cinematic cloudscape over the ocean",
    "async": true
  }'
```

### 查询任务结果
```bash
curl http://localhost:8000/v1/videos/generations/<job_id> \
  -H "Authorization: Bearer bing-demo-key"
```

### 流式返回说明

当前 Bing 视频接口**无法 SSE / chunked 流式输出**，也不返回 `text/event-stream`。

因此本项目对外也采用同样的稳定语义：

- `POST /v1/videos/generations`：提交任务
- `GET /v1/videos/generations/<job_id>`：轮询任务状态与结果

推荐调用方式：

1. 创建任务时传 `"async": true`
2. 从响应中拿到 `id`
3. 周期性查询 `/v1/videos/generations/<job_id>`
4. 当 `status` 变为 `succeeded` 后读取结果视频地址

示例：

```bash
curl http://localhost:8000/v1/videos/generations \
  -H "Authorization: Bearer bing-demo-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sora-v2-fast",
    "prompt": "a panda riding a bicycle in the rain",
    "async": true
  }'
```

典型响应：

```json
{
  "id": "job_xxx",
  "object": "video.generation",
  "created": 1740000000,
  "model": "sora-v2-fast",
  "status": "queued"
}
```

随后轮询：

```bash
curl http://localhost:8000/v1/videos/generations/job_xxx \
  -H "Authorization: Bearer bing-demo-key"
```

完成后响应示例：

```json
{
  "id": "job_xxx",
  "object": "video.generation",
  "created": 1740000000,
  "model": "sora-v2-fast",
  "status": "succeeded",
  "result": {
    "url": "https://...mp4",
    "thumbnail_url": "https://...jpg",
    "mime_type": "video/mp4",
    "aspect_ratio": "16:9"
  }
}
```

如果传 `"async": false`，服务端会等待任务完成后一次性返回 JSON 结果；这仍然是**阻塞式单次响应**，不是流式输出。
本项目提供视频生成前端：test-video.html

---

## ⚠️ 重要限制
- 依赖有效 Bing 账户 Cookie 与当前会话状态
- `skey` 为短期字段，每次卡片流程动态获取
- 生成视频可能含水印（不移除）
- 12s 时长仅实验性探测，实际产出仍为 8 秒

---

## 🔐 获取 Bing Cookie

### Chromium 内核（Edge/Brave/Opera）
- 打开 https://bing.com/
- F12 → Console
- 执行：
  ```js
  cookieStore.get("_U").then(result => console.log(result.value))
  ```
- 复制 `_U` 用于导入

### 推荐的 Bing 会话提取脚本
如果控制台中没有 `_EDGE_S`，但在 Application/Cookies 可见，可先使用可见 Cookie 作为初始会话，让后端刷新 Bing 会话状态。手工从已登录页面提取可用脚本：

```js
Promise.all([
  "_U",
  "_EDGE_S",
  "SRCHUSR",
  "SRCHUID",
  "SRCHD",
  "MUID",
  "MUIDB",
  "ANON",
  "WLS"
].map(name => cookieStore.get(name).then(v => [name, v?.value || null])))
  .then(entries => entries.filter(([, v]) => v))
  .then(entries => entries.map(([k, v]) => `${k}=${v}`).join("; "))
  .then(console.log)
```

**完整 Cookie 集合（图生视频必需）**

图生视频的图片上传需要 `.MSA.Auth`（httpOnly Cookie，JavaScript 无法读取）。获取方法：

1. 在已登录的浏览器中打开 https://www.bing.com/images/create/ai-video-generator
2. F12 → Network 面板
3. 刷新页面
4. 点击第一个发往 `bing.com` 的请求
5. 在 Headers 标签中找到 `cookie:` 请求头
6. **复制整个 cookie header 的值**，粘贴到管理后台导入账号时的 `cookie_header` 字段

管理后台支持直接粘贴完整的 cookie header 字符串，会自动解析所有键值对。

图生视频上传当前使用**真实浏览器自动化**完成，以复用浏览器上下文与会话状态。运行图生视频前请确保环境具备：

- Node.js
- `playwright-core`（已在 `package.json` 中声明）
- 本机 Chrome / Edge 可执行文件
- 可用代理（如你的 Bing 访问依赖代理）

浏览器路径可通过环境变量 `CHROME_PATH` 指定，代理可通过 `BROWSER_PROXY` 指定。

**最小 Cookie 集合（仅文生视频）**
`_U + _EDGE_S + SRCHUSR + SRCHUID + SRCHD + MUID + MUIDB + ANON + WLS`

如果浏览器只暴露 `_U` 与少量会话 Cookie，也可以先导入轻量 Cookie 头到后台，并使用 `刷新会话` 动作触发 Bing 颁发 `_EDGE_S`。

多账户导入时，后台支持批量会话刷新流程，用于批量补全 `_EDGE_S` 与快模式配额状态。

推荐工作流：
1. 导入轻量 Bing Cookie
2. 执行批量账户准备
3. 后端刷新 Bing 会话并补全 `_EDGE_S`
4. 后端刷新快模式配额状态
5. 再将账户加入路由

### Firefox
- 打开 https://bing.com/
- F12 打开开发者工具
- 进入存储（Storage）面板
- 展开 Cookies
- 选择 `https://bing.com` 的 Cookie
- 复制 `_U` 的值

---

## 🧭 路由与并发建议
- 文生视频：使用轻量 `_U` 会话即可
- 图生视频：需要完整 Cookie（含 `.MSA.Auth`），从浏览器 Network 面板复制完整 cookie header
- 每次上传前动态派生 `SID`
- 账户池路由 + 单账户并发限额 + 故障转移


---

## ⚙️ 运行时设置
管理后台新增 **系统设置** 标签页，保存后会写入 `data/settings.json`，并立即生效，无需重启服务。

支持修改：
- OpenAI API Key（逗号分隔）
- 全局代理
- 轮询间隔 / 超时参数（快/慢分开）
- 图生视频上传模式（浏览器优先 / 仅浏览器上传）
- 图生视频浏览器上传并发上限

---

## 🎯 支持的模型

| 模型 ID | 画幅 | 模式 | 说明 |
| --- | --- | --- | --- |
| `sora-v2-fast` | 16:9 | fast | 文生视频 / 图生视频 |
| `sora-v2-slow` | 16:9 | slow | 文生视频 / 图生视频 |
| `sora-v2-landscape-fast` | 16:9 | fast | 文生视频 / 图生视频 |
| `sora-v2-landscape-slow` | 16:9 | slow | 文生视频 / 图生视频 |
| `sora-v2-portrait-fast` | 9:16 | fast | 文生视频 / 图生视频 |
| `sora-v2-portrait-slow` | 9:16 | slow | 文生视频 / 图生视频 |
