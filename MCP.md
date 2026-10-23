# 蔓藤 AI MCP 使用指南

配置一次，之后就可以在 **Claude Code / Codex** 等支持 MCP 的客户端里，用自然语言完成图片生成、声音克隆、音视频处理、数字人视频和 PPT 生成。

你不需要记工具名。直接描述要做什么，客户端会自动选择工具、上传素材、提交任务并等待结果。

---

## 1. 5 分钟接入

### 1.1 获取访问 token

登录蔓藤 AI 网页版，进入 **API密钥** 页面，复制页面上的 token。

token 与你的平台账号绑定。账号是否可用、当前权益等信息，可在账户页面查看。

### 1.2 配置 Claude Code

在启动 Claude Code 的同一个终端里设置环境变量：

```bash
export MANTENG_TOKEN='粘贴你的 token'
```

然后添加 MCP 服务：

```bash
claude mcp add --transport http manteng https://api.mindtechassist.com/mcp \
  --header "Authorization: Bearer $MANTENG_TOKEN"
```

### 1.3 配置 Codex

编辑 `~/.codex/config.toml`：

```toml
[mcp_servers.manteng]
url = "https://api.mindtechassist.com/mcp"
bearer_token_env_var = "MANTENG_TOKEN"
```

设置环境变量后重启 Codex：

```bash
export MANTENG_TOKEN='粘贴你的 token'
```

### 1.4 验证是否接入成功

重启客户端后，直接说：

```text
用蔓藤 MCP 生成一张戴宇航头盔的狐狸图片，3D 国风，电影光效。
```

客户端能提交任务并持续查询进度，就说明配置成功。

---

## 2. 可以直接这样说

| 想做什么 | 可以直接对客户端说 |
|---|---|
| 生成图片 | “生成一张产品海报，主体是透明香水瓶，极简高级感，竖版。” |
| 克隆声音 | “这是我的录音，帮我克隆成音色，名字叫‘我的声音’。” |
| 语音合成 | “用‘我的声音’朗读这段文案，并给我音频文件。” |
| 人声分离 | “把这段视频里的人声和背景音分开，保留人声。” |
| 换音色 | “把这段音频里的声音换成另一个音色。” |
| 视频处理 | “把这段视频剪成 30 秒，去掉开头 5 秒，并增强画质。” |
| 视频翻译 | “把这段中文视频翻译成英文配音，保留原字幕样式。” |
| 数字人视频 | “用这张形象和这个音色，生成一段 30 秒口播视频。” |
| 生成 PPT | “根据这份文档生成一份 10 页、适合投资人的 PPT。” |
| 动效短片 | “根据这些产品素材，出一个 30 秒动效短片方案。” |

如果客户端需要更多信息，它会继续向你要文件、链接或参数。按提示补充即可。

---

## 3. 最常见的工作流

### 3.1 生成图片

```text
帮我生成一张 9:16 的手机壁纸：夜晚的赛博朋克城市，霓虹灯，电影感。
```

提交后客户端会拿到任务 ID，并自动查询到完成。最终结果是一个图片地址，可直接打开或下载。

### 3.2 克隆声音并朗读文案

```text
我上传一段录音。先帮我克隆成音色，名字叫“我的声音”；
等克隆完成后，用它朗读“欢迎来到蔓藤 AI”，最后给我音频文件。
```

推荐顺序：

1. 上传音频，得到素材引用。
2. 调用声音克隆，得到 `recordId`。
3. 等待任务完成，拿到新的 `voiceId`。
4. 用 `voiceId` 合成语音。
5. 等待语音任务完成，返回音频地址。

克隆未完成时不能直接用于合成，客户端应继续等待。

### 3.3 处理音频或视频

先把本地文件或公开链接交给客户端，再说需求即可：

```text
把这段采访视频里的人声分离出来，背景音乐单独保存。
```

| 需求 | 客户端会使用的工具 |
|---|---|
| 人声分离 | `separate_vocals` |
| 换音色 | `change_voice` |
| 音频剪辑、转格式 | `edit_audio` |
| 视频剪辑 | `edit_video` |
| 换脸 | `face_swap` |
| 画质增强 | `enhance_media` |
| 抠图、图片处理 | `image_process` |
| 视频翻译 | `translate_video` |
| PPT 数字人讲解视频 | `ppt_video` |

### 3.4 生成可编辑 PPT

```text
根据这份文档，做一份 10 页的融资路演 PPT，风格简洁，重点突出市场规模和产品优势。
```

提交后等待任务完成即可。若需要逐页修改或导出可编辑文件，请打开网页版 PPT 编辑台继续处理。

### 3.5 数字人和动效短片

- 数字人讲解视频：准备一张形象图和一个音色，直接描述视频主题与时长。
- 动效短片：提供产品素材、卖点和目标时长，客户端会先出方案，再继续渲染。
- `animation_video_plan` 使用的声音需要是用户自己克隆的音色。

不确定有哪些音色或形象时，可以直接说：

```text
先列出我现有的音色和形象，让我选一个。
```

---

## 4. 客户端应该怎样等任务

大部分生成任务都是异步的。提交后通常返回 `recordId`，而不是立刻返回成品。

```text
提交任务 → recordId
        ↓
get_task {record_id, wait_seconds: 20}
        ↓
taskSucceeded=true → 使用 resultUrl
taskSucceeded=false → 把 errorMessage 告诉用户
nextAction=confirm → 先询问用户，再 confirm_task
nextAction=blocked → 请用户补充信息
```

判断任务是否完成，以这些字段为准：

| 字段 | 含义 |
|---|---|
| `status` | `QUEUE` / `PROCESSING` / `COMPLETED` / `FAILED` / `BLOCKED` |
| `terminal` | 是否已经结束 |
| `taskSucceeded` | 只有 `COMPLETED` 才是 `true` |
| `nextAction` | `poll` / `done` / `failed` / `confirm` / `blocked` |
| `retryAfterSeconds` | 下次查询建议等待的秒数 |

提示：`get_task` 调用成功不等于生成任务成功，必须继续检查 `taskSucceeded`。

可以给客户端加一句总要求：

```text
提交生成任务后，请自动轮询 get_task 直到任务结束；
不要在任务未完成时告诉我已经完成。
```

---

## 5. 上传素材

### 5.1 本地文件

直接告诉客户端文件路径，例如：

```text
用 /Users/me/Desktop/voice.m4a 这份录音克隆音色。
```

客户端会先申请上传票据，上传文件，再把素材引用交给生成工具。

### 5.2 已有公开链接

```text
用这个地址作为参考音频：https://example.com/voice.m4a
```

链接必须可公开访问，并且属于平台允许的域名。

### 5.3 上传限制

- 一张票据只能上传一个文件，重复上传会覆盖同一个对象。
- 上传票据默认 10 分钟有效；过期后重新申请。
- 文档最大 20MB，音视频和图片最大 200MB。
- 每个 token 每天最多申请 10 张票据。
- 没有 shell 或网络权限时，让用户提供公开链接即可。

---

## 6. 工具清单

以客户端通过 `tools/list` 实际返回为准。

**生成**

`generate_image`、`create_deck`、`animation_video_plan`、`synthesize_speech`

**素材处理**

`change_voice`、`separate_vocals`、`edit_audio`、`edit_video`、`face_swap`、`enhance_media`、`image_process`、`translate_video`、`ppt_video`

**克隆与资产**

`clone_voice`、`clone_avatar`、`list_voices`、`list_avatars`

**文档与表格**

`read_attachment`、`query_table`

**MCP 专属**

| 工具 | 作用 |
|---|---|
| `create_upload_ticket` | 生成本地上传票据，并返回可直接执行的 curl |
| `import_media` | 登记已有素材链接，返回 `ref` |
| `list_media` | 查看当前会话已导入的素材 |
| `get_task` | 查询进度、产物、失败原因和下一步 |
| `confirm_task` | 确认等待用户确认的任务 |
| `query_tasks` | 查看当前会话最近任务 |

需要在网页版继续完成的能力：可编辑 PPT 的逐页编辑与导出、字幕编辑。MCP 终端不提供任务卡或页面按钮，客户端应直接返回结果链接和操作说明。

---

## 7. 使用限制

| 项 | 限制 |
|---|---|
| 上传 | 文档 20MB，其余 200MB；票据 10 分钟；每天最多 10 张；一票一文件 |
| MCP 会话 | 2 小时；最多 50 份素材；最多 500 次真实工具调用 |
| 轮询 | `wait_seconds` 取 0–30；同时等待请求有上限 |
| 请求频率 | 按 IP 限流；触发后稍后重试 |
| 终端能力 | 不能逐页编辑或导出可编辑 PPT；字幕编辑需网页版 |

---

## 8. 常见问题

- **401**：`Authorization` 或 `x-api-key` 缺失、过期或无效；重新复制 token 并更新环境变量。
- **404 + `-32001`**：`Mcp-Session-Id` 已过期或不属于当前账号；重新初始化会话。
- **上传返回 403 / `InvalidPolicyDocument`**：上传票据过期；重新申请。
- **`isError=true`**：查看 `errorMessage`，通常是参数、账户状态或上游服务问题。
- **任务一直 `QUEUE`**：服务队列繁忙或 worker 暂时不可用；继续按 `retryAfterSeconds` 查询。
- **任务 `FAILED`**：把 `errorMessage` 原样告诉用户，不要当成成功。
- **`nextAction=confirm`**：先展示 `checkpoint.reason`，用户同意后再调用 `confirm_task`。

---

## 9. 安全建议

- token 就是账号凭证，不要发到公开仓库、聊天记录或日志里。
- 克隆声音、克隆形象、确认继续执行前，先向用户说明将要生成的内容。
- 客户端支持逐次工具确认时，建议保持开启，尤其是批量任务和长视频任务。

---

## 附录：协议参考

普通用户只需要完成第 1 节的配置。以下内容供需要调试 MCP 的开发者使用。

### initialize

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
  "protocolVersion":"2025-06-18",
  "capabilities":{},
  "clientInfo":{"name":"claude-code","version":"1.x"}
}}
```

响应头包含 `Mcp-Session-Id`。随后发送 `notifications/initialized`。

### tools/list

```json
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
```

### tools/call

```json
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{
  "name":"generate_image",
  "arguments":{"prompt":"一只狐狸","aspect":"portrait"}
}}
```

成功响应：

```json
{"jsonrpc":"2.0","id":3,"result":{
  "content":[{"type":"text","text":"已提交图片生成…\nrecordId=123（用 get_task 查询进度）"}],
  "structuredContent":{"ok":true,"toolKey":"generate_image","recordId":123},
  "isError":false
}}
```

工具执行失败同样走 `result`，但 `isError=true`，原因在 `structuredContent.errorCode/errorMessage`。

### 协议级错误

| code | 含义 | HTTP |
|---|---|---|
| -32700 | body 不是合法 JSON | 400 |
| -32600 | 不是合法 JSON-RPC 请求 | 400 |
| -32601 | 方法不支持 | 200 |
| -32602 | 参数错误或未知工具 | 200 |
| -32603 | 服务端异常 | 500 |
| -32001 | `Mcp-Session-Id` 不存在或不属于当前用户 | 404 |
