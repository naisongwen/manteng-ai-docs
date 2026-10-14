# 可编辑 PPT：调研、设计与实施方案

> 本文是内部设计文档。产品对外的说明文档另出。
> 调研时间：2026-10

---

## 0. 一句话

在现有「数字人讲 PPT」之外，新增一条独立产品线：

**输入主题 / 文档 → 生成整套幻灯片 → 网页里自由编辑 → 导出可改字的 `.pptx` / `.pdf`**

跟现有产品线的区别：现有那条是「PPTX 进 → MP4 出」，这条是「内容进 → PPTX 出」。两者共用后端骨架和 worker，但不共用编辑器。

### 本版不做

- **TTS / 讲稿配音**——不做
- **视频导出（MP4 / 数字人讲解）**——不做
- 页面上保留 `notes`（备注）字段做纯文本存续，但不接合成链路

范围就是两件事：**编辑** 和 **导出 pptx / pdf**。

---

## 1. 调研：开源方案现状

### 1.1 三条技术路线

| 路线 | 代表方案 | 视觉质量 | 真可编辑 | 稳定性 |
|---|---|---|---|---|
| **A. HTML 自由排版 → DOM 量测 → 原生对象** | `Design-Arena/html-to-pptx`、Anthropic `html2pptx`、DeepPresenter 自由模式 | 最高 | 中（转换有损） | 低，排版易错位 |
| **B. 模板槽位 → 结构化 spec → 原生对象** | PPTAgent 模板模式、Presenton 标准管线 | 中 | 最高 | 高 |
| **C. 生图 → 逆向拆图层** | `gpt-image2-ppt-skills` editable 模式 | 最高 | **假** | 中 |

**路线 C 单独说明。** 它听着最香（图好看又"可编辑"），但可编辑性是**事后补出来的**：先生成整页图，再拆成「干净底图 + 文本框 + 形状 + 图片层」叠回去。字能选了，但和底图上的原字会错位、产生重影，放大就露。它还需要一台 LibreOffice/Keynote 做回渲 QA。适合「要好看、能改几个字」，不适合当服务底座。

### 1.2 两个必须知道的许可陷阱

**① Anthropic 官方 `pptx` skill 不是开源。**

`anthropics/skills` 根目录故意不放 LICENSE，许可下沉到各 skill 子目录。`document-skills` 下的四本（docx / pdf / pptx / xlsx）是 **source-available, not open source**，LICENSE.txt 明确禁止商用、复制、衍生、分发、逆向。**做商用服务不能碰。** 同仓的 `example-skills` 才是 Apache 2.0，但不含 pptx 能力。

**② Presenton 的导出核心闭源。**

主仓 Apache-2.0、9k+ star，但 `scripts/sync-presentation-export.cjs` 会从 GitHub Releases 拉一个**预编译 tarball** 进 `presentation-export/node_modules`。真正的 binary→OOXML 转换在 `@presenton/export-core`，源码仓 `presenton/presenton-export` 只有一个 README。

「可编辑度」恰好是被藏起来的那一环。**不能当底座，只能当对标。**

### 1.3 调研结论

**不引入任何外部仓库。** 理由见第 2 节——我们自己已经有 80% 的零件，而唯一缺的那一环（矢量页面模型 + 导出器）外部方案要么闭源、要么许可受限、要么架构不匹配（我们是绝对定位画布，不需要 DOM 量测）。

外部仓库降级为**参考清单**：

| 仓库 | 许可 | 参考什么 |
|---|---|---|
| `presenton/presenton` | Apache-2.0（导出核心除外） | `PptxPresentationModel` 的 TS/Python 双端同构设计 |
| `icip-cas/PPTAgent` | Apache-2.0 | generate→render→review→fix 自省环 |
| `Design-Arena/html-to-pptx` | MIT | 仅当将来要做「LLM 自由排版 HTML」时才用 |

---

## 2. 现状盘点：我们有什么

### 2.1 可复用的

| 零件 | 位置 | 复用方式 |
|---|---|---|
| `python-pptx` | worker 依赖（`sse/utils/ppt_utils.py`） | 导出器直接用 |
| LibreOffice headless | `ppt_utils.py:148-153` | 回渲 QA、PDF 导出 |
| 页级派发 / 账单 / OSS | `PptRenderBatchServiceImpl`、`ppt_page_render` 表 | 骨架照抄 |
| 配额与冻结 | `interactive_record` + `creditsOf` | 直接接 |
| 拖拽/缩放交互与坐标换算 | `ppt-detail/index.vue:1201,1215,1631-1647,1709-1710` | 新编辑器搬 |
| 逐像素验收方法 | `shotcraft` 那条线 | QA 环照搬 |

### 2.2 不能复用的：现有编辑器

**`src/views/ppt-detail/index.vue` 不是 PPT 编辑器，是「数字人形象版式标注器」。** 证据：

1. **底图是栅格 PNG。** `drawBaseImage`（`:1399-1425`）把 `currentPageImageUrl` 全屏 `drawImage`——那是上传 PPTX 经 soffice→PDF→PyMuPDF 渲出来的整页图。`isCanvasBaseLayer`（`:1429`）的判据就是「图层 URL == 页面底图 URL」。
   → **结构上不可能导出可编辑 pptx**，最多得到「一张整页图 + 几个框」。
2. **Layer 没有排版模型。** 全部字段只有 `type/content/renderUrl/x/y/width/height/isAvatar/pageIndex`（`:47-69`）。没有字号、字体、颜色、对齐、行距、项目符号、透明度、旋转、z-index。图层类型只有 `text | image | video`。
3. **文字图层是残的。** `drawTextLayer`（`:1436`）写死 `16px Arial` 纯黑、无换行、无样式。好看的排版在 worker 的 ffmpeg overlay 里。
4. **没有页面 CRUD。** 全文件搜不到 addPage / deletePage / duplicatePage / movePage。页数由上传的 pptx 决定。
5. **API 里零 pptx 导出。** `ppt.ts` 19 个接口，产物一律是 MP4 / 音频。

**结论：新建模块，不改造。** 现有编辑器与目标模型正交。

---

## 3. 设计考虑点

以下七条是**改起来代价最大**的决策，先定死。

### 3.1 不让 LLM 直接吐坐标

「网页里自由编辑」不等于「LLM 自由排版像素」。让模型直接产出每个文本框的绝对坐标，出来必是垃圾：重叠、出界、字号乱。

正确分工：

- **LLM 只做两件事**：① 出内容（标题 / 要点 / 数据 / 配图语义）；② 选版式意图（「这页是大标题+三卡」还是「左图右文」）。
- **确定性排版器把版式意图算成绝对坐标**，按栅格落成 `Layer[]`。

这跟 `shotcraft` 是同一个 DNA——LLM 挑组件、填 props，坐标由排版器一处定死。

出口仍是绝对坐标的 `Layer[]`，用户接着随便拖。**确定性只作用于初稿，不限制后续编辑。**

### 3.2 Layer 存「语义角色 + token」，不存绝对样式

```diff
- { "fontSize": 24, "color": "#1a1a1a", "fontFamily": "PingFang SC" }
+ { "role": "title", "style": { "colorToken": "fg-primary", "sizeStep": 3 } }
```

**这是「一键换风格」能不能做成的分水岭。** 存绝对像素的话，用户换风格时你得从像素反推语义角色——推不准就换不干净。

用户手改过的地方存 `userOverrides` 覆盖层，就是 CSS 的层叠：**token 基线 + 用户覆盖**。

推论：**组件里不许硬编码任何颜色和字号，只能引用 token。**

### 3.3 模板 = 风格 × 组件，不做「整页模板」

| 层 | 是什么 | 数量级 | 谁在变 |
|---|---|---|---|
| **风格** | 整体视觉语言：商务蓝 / 科技暗色 / 杂志风 / 手绘风 | 5–10 套 | 用户选 |
| **主题 token** | 配色、字体对、圆角、间距刻度、阴影层级 | 每套风格一套 | 风格决定 |
| **版式组件** | 标题、三卡、对比、时间线、数据页、引用、结尾 | 25–40 个 | 排版器选，用户可覆盖 |

用户看到的「模板」= 风格 × 组件。10 套风格 × 30 个组件，视觉上像几百套模板，**实际维护量是 10 + 30**。

**不做「模板 A 的第 3 页长这样」那种整页模板**：每加一套风格要重画所有页面，维护量变成 风格数 × 页数，且用户改不动。这正是 Presenton 那条路，长期会变成谁都不敢碰的资产。

### 3.4 字体：生成端锁死 + 服务器装字体

**现状（已核实 2026-10-14）**

| 机器 | 中文字体数 | 有哪些 |
|---|---|---|
| worker 服务器 `aiserver0` | **1** | 只有 `DroidSansFallbackFull` |
| 开发机 macOS | 99 | 苹方、宋体、黑体… |

字体问题对**不同环节的影响完全不同**，必须分开处理：

| 环节 | 服务器字体够不够 | 原因 |
|---|---|---|
| **导出 `.pptx`** | ✅ **不依赖** | 只写字体名，不渲染。由用户端的 PowerPoint 解析 |
| **生成侧回渲 QA** | ❌ 不够 | 渲出来是 Droid fallback，跟用户看到的不是一个字形 |
| **导出 `.pdf`** | ❌ 不够 | PDF 在服务器渲染，字体就是 Droid |

→ **必须给服务器装字体**，否则 PDF 导出不可用、QA 无意义。

**选什么字体：Noto Sans CJK SC（= 思源黑体，OFL）**

- 免费、可自由分发、**可嵌入**
- 服务器装了之后，PDF 导出和 QA 回渲都准确
- 用户端没装会 fallback 到系统默认中文黑体（微软雅黑 / 苹方）——**同为黑体，观感接近**，是代价最小的选择

**为什么不做字体嵌入**：技术上要手写 OOXML，真正的墙是**授权**——可嵌入的中文字体只有思源系列（OFL）这类，苹方、微软雅黑不允许嵌入。

**代价**：牺牲用户的字体自由度。生成端只允许用白名单字体，用户不能改。**这是主动放弃，要写进产品说明。**

### 3.5 回渲 QA 放生成侧，且只查结构性错误

导出链路里最重的一步是 LibreOffice 回渲（单页 0.5–2s，40 页要 1–2 分钟）。

- **生成初稿时**回渲一次，验证布局对不对（低频）
- **导出时**只把已经验证过的 IR 落成 pptx，纯 `python-pptx`，秒级，不需要 soffice

**但它有个系统性误差**：服务器即使装了 Noto Sans CJK，用户端用的也可能是微软雅黑/苹方，**字形和度量本就不同**。所以 QA 的判据只能是**结构性错误**：

| QA 能查 | QA 查不了 |
|---|---|
| 元素重叠 | 文字实际断行位置 |
| 超出页面边界 | 行距的像素级差异 |
| 文本溢出容器 | 用户端的最终观感 |
| 缺图 / 图片比例失真 | |

**唯一 100% 保真的交付格式是 PDF**（服务器渲染，不依赖用户字体）。PPTX 的保真度天花板就是「结构正确 + 字体 fallback 后观感接近」。

### 3.5 回渲 QA 放生成侧，不放导出侧

导出链路里最重的一步是 LibreOffice 回渲（单页 0.5–2s，40 页要 1–2 分钟）。

- **生成初稿时**回渲一次，验证布局对不对（低频）
- **导出时**只把已经验证过的 IR 落成 pptx，纯 `python-pptx`，秒级，不需要 soffice

### 3.6 导出走独立队列，别和渲染共用慢池

现有 worker：快池 1 线程、慢池 2 线程，`ppt_render` / `ppt_parse` 都在慢池，慢任务超时 90 分钟。

如果一个 90 分钟的数字人渲染正占着慢池线程，用户点「导出 PPTX」就要排在它后面。**秒级任务被分钟级任务堵死，是必现的产品事故。**

→ 新增**导出池**（或在快池单开），并发硬卡 1–2。反正秒级任务排队无感，但能防住一批人同时导出把 soffice 进程堆爆。

### 3.7 导出是单向投影

「网页里编辑」和「PowerPoint 里编辑」是两件事。前者编辑 IR，后者编辑 pptx。

- **只做单向导出**，不做双向同步——用户在 PowerPoint 里改完传回来，网页编辑器看不懂，这是无底洞。
- 「用户上传自己的 PPTX 反解成可编辑图层」也不做，或放到很后面。现有 soffice→PDF→PNG 只出栅格页，反解要啃 pptx 逆向解析，工作量是另一个量级。

---

## 4. 能力边界

### 4.1 能做到的

**生成**
- 输入：主题 / 大纲 / 上传文档（doc、pdf、txt）→ 一次出整套
- 版式：由组件库决定，排版器按内容量自动选
- 配图：AI 生图 / 图库 / 用户上传
- 风格：一键换配色与字体方案

**编辑**
- 自由画布：拖拽、缩放、旋转、栅格吸附、参考线
- 图层：文字、图片、形状、图标、线段
- 页：新增 / 删除 / 复制 / 排序
- 文本：字号、颜色、粗斜体、对齐、行距、项目符号
- 图片：上传、裁剪、圆角、替换
- 图层树：z-order、锁定、隐藏、编组
- 撤销重做、对齐分布、备注栏

**导出**
- `.pptx`：原生文本框 + 形状 + 图片，**能改字、能挪框、能换图**
- `.pdf`、每页 PNG

### 4.2 打折扣的（能做，但别期待满分）

| 能力 | 现实 | 原因 |
|---|---|---|
| 复杂视觉（渐变、阴影、混合模式、图标字体） | 降级成**图片层**。看着对，但在 PowerPoint 里改不了 | PPTX 没有对应原语 |
| 图表 | 简单图表可做原生 chart；定制样式退回图片 | python-pptx 图表 API 有限 |
| 表格 | 基础表格可以；合并单元格、条件格式、排序不行 | 同上 |
| 导出保真 | **「看着差不多」，不是逐像素相同** | PowerPoint 与浏览器对换行、行距、字距的计算本就不同 |
| 字体 | 只能给白名单 | 不锁死就必然错位 |
| 主题 / 母版 | 能做 token 级主题，**做不了 PowerPoint 的 Slide Master 继承** | 概念映射不完整 |

### 4.3 做不到的

**A. PPTX 格式的天花板（再努力也绕不过）**

1. **动画与切换效果** —— python-pptx 和 PptxGenJS 都不支持创建
2. **SmartArt** —— 同上
3. **字体嵌入** —— 授权问题（见 3.4）
4. **公式（OMML）**

→ 这四项只有一条出路：**Aspose.Slides（商业授权）**。按部署收费。要不要为了动画买它，是纯商业决定。

**B. 架构上明确不做的**

1. 用户上传 PPTX 反解成可编辑图层
2. 与 PowerPoint 双向同步
3. 多人实时协作（技术上能做，单机也扛得住，但要加 CRDT + 长连接，是纯工作量问题，可后加）

### 4.4 产品文案的红线

**别写「导出的 PPT 可以随意编辑」。** 用户一试形状和动画就会觉得被骗。

准确说法：**「导出的 PPTX 里，文字都能直接改」。**

---

## 5. 实施方案

### 5.1 模块划分

| 模块 | 位置 | 工作 |
|---|---|---|
| 编辑器页面 | `manteng-ai/src/views/deck/`（新建） | 矢量画布编辑器 + 风格/版式面板 |
| 渲染层 | `manteng-ai/src/views/deck/components/` | 组件库（25–40 个版式组件） |
| 主题 | `manteng-ai/src/.../themes/` | token 表，5–10 套 |
| 后端 | `aigc-backend/src/main/java/org/aigc/deck/`（新建） | 增删改查 + 生成编排 + 导出派发 |
| 排版器 | `aigc-backend` 或 worker | LLM 内容 + 版式意图 → `Layer[]` |
| 导出器 | `artificial-intelligence-notebook/sse/handlers/deck_handler.py`（新建） | `IR` → python-pptx |
| QA 环 | worker | 生成侧回渲比对 |

### 5.2 核心数据模型

**设计空间统一 1920×1080**（不用现有编辑器的 768×432）。换算干净：导出时 `pt = px / 2`。

```jsonc
// Deck
{
  "id": "...",
  "title": "...",
  "themeId": "business-blue",
  "designW": 1920, "designH": 1080,
  "slides": [ /* Slide */ ]
}

// Slide
{
  "id": "...",
  "index": 0,
  "layout": "three-cards",          // 版式组件名，用于「换版式」
  "background": { "token": "bg-base" },   // 或 { "image": "oss://..." }
  "notes": "这一页的讲稿",
  "layers": [ /* Layer */ ]
}

// Layer —— 注意这里只有 token 引用，没有绝对样式
{
  "id": "...",
  "kind": "text" | "image" | "shape",
  "role": "title" | "body" | "caption" | "card-title",
  "box": { "x": 120, "y": 80, "w": 800, "h": 120 },
  "rotation": 0,
  "z": 3,
  "locked": false,
  "visible": true,

  "style": {                         // 全部是 token / 刻度引用
    "colorToken": "fg-primary",
    "fontToken": "font-display",
    "sizeStep": 3,                   // 字号刻度，不是 px
    "weight": "bold",
    "align": "left",
    "lineHeight": 1.4
  },
  "overrides": { },                  // 用户手改，覆盖 token

  "text": "...",                     // kind = text
  "src": "oss://...",                // kind = image
  "shape": "rounded-rect"            // kind = shape
}
```

**主题 token 表**（每套风格一份）：

```jsonc
{
  "id": "business-blue",
  "colors": {
    "bg-base": "#ffffff", "bg-accent": "#f1f5f9",
    "fg-primary": "#0f172a", "fg-secondary": "#64748b",
    "brand": "#2563eb", "brand-fg": "#ffffff"
  },
  "fonts": { "font-display": "Source Han Sans", "font-body": "Source Han Sans" },
  "scale":  { "size-steps": [14, 18, 24, 32, 44, 60], "space-steps": [8,16,24,40,64] },
  "shape":  { "radius": 12, "shadow": "sm" }
}
```

### 5.3 落库

沿用现有「整体保存一页」的模式（同 `/ppt/update_page` 存 `material` 数组）：

```
ppt_deck   (id, user_id, title, theme_id, status, version, created_at, updated_at)
ppt_slide  (id, deck_id, idx, layout, background, notes, layers jsonb, version)
```

- `layers` 存 `jsonb`，整页保存，不拆表。编辑器自动保存本来就是一页一提交。
- 写 jsonb 必须显式转型 `?::jsonb`（连接串无 `stringtype=unspecified`）。
- 用 `SimpleJdbcInsert` 时**必须指定 `usingColumns`**，否则缺列会被绑成 NULL，DDL 默认值不生效。
- DDL 无迁移框架，需手动 `ssh + psql` 应用。

### 5.4 接口

```
POST   /deck/create            生成整套（异步，返回 taskId）
POST   /deck/get               取整套
POST   /deck/update_slide      保存单页（整页 layers）
POST   /deck/add_slide         新增页
POST   /deck/delete_slide      删除页
POST   /deck/reorder_slides    页排序
POST   /deck/apply_theme       换风格（只换 themeId，前端重渲染）
POST   /deck/relayout_slide    单页换版式（重跑排版器）
POST   /deck/export            { deckId, format: pptx | pdf | png }
GET    /deck/themes            风格列表
```

### 5.5 导出映射

| IR | PPTX |
|---|---|
| `box.x/y/w/h`（1920 设计 px） | `Inches(px / 144)` —— 1px = 1/144 in，1920px = 13.333in |
| `sizeStep` → `scale.size-steps[step]` | `Pt(px / 2)` |
| `kind: text` | `add_textbox`，关闭 autofit，`word_wrap = True` |
| `kind: image` | `add_picture` |
| `kind: shape` | `add_shape`（圆角矩形 / 椭圆 / 直线） |
| `colorToken` | 查主题表 → `RGBColor` |
| `fontToken` | 写字体名（白名单，见 3.4）。同时设 latin + eastasian，主题字体也一并设 |
| 复杂视觉层 | 降级为图片层 |

**两个 python-pptx 的坑（M1 实测踩到）**

1. **中文字体要写三处。** `run.font.name` 只写 `<a:latin>`，中文会退回主题默认字体。必须额外写 `<a:ea>` / `<a:cs>`（东亚字体是 `<a:ea>`），且顺序是 latin → ea → cs。

2. **空 `<a:effectLst/>` 挡不住 LibreOffice 的投影。** python-pptx 的 `add_shape` / `add_connector` 会带一个引用主题的 `<p:style>`，里面有 `<a:effectRef idx="2">`（主题投影）。按 OOXML 规范，显式的空 `<a:effectLst/>` 应该覆盖它，PowerPoint 也确实认 —— **但 LibreOffice 不认**，照样渲出投影。而回渲 QA 和 PDF 导出都走 LibreOffice，所以必须把 `<p:style>` 整个删掉。`lnRef` / `fillRef` 同理，会让「没设描边」的形状长出一条主题色的边。

### 5.6 里程碑

| 阶段 | 内容 | 验收 |
|---|---|---|
| **M1** | IR schema + 导出器（手工 JSON → pptx） | 导出的 pptx 与预览图逐页对照，肉眼无差异 |
| **M2** | 编辑器（矢量画布 + 存/取） | 能建页、拖框、改字、存回、刷新后一致 |
| **M3** | 生成端（LLM 内容 + 版式意图 → 排版器） | 给主题出一套，无重叠/出界/字号乱 |
| **M4** | 主题与组件库（风格 × 组件） | 一键换风格，内容不动，每套都好看 |
| **M5** | QA 回环 + 验收矩阵（组件 × 风格全量渲染比对） | 改一次 token 全量重渲无回归 |
| **M6** | PDF / PNG 导出、配额计费、导出池 | 导出不排队；扣费与报价一致 |

**先做 M1。** 导出保真度是整条链路风险最高的地方，先用手工 JSON 把「IR → pptx → 回渲 → 对照」这条路走通，再往上叠东西。

### 5.7 单机资源

| 动作 | 频率 | 成本 |
|---|---|---|
| 保存图层 | 每次拖拽 | 一次 JSON 写库 |
| 导出 pptx（40 页） | 每次点导出 | 秒级，纯 CPU |
| 回渲 QA | 每次生成初稿 | 20–80s，CPU + 磁盘 |
| 每页 PNG（150 DPI） | — | 1–3MB/页，走 OSS，临时文件要清 |

- **零 GPU 依赖**，不跟 HeyGem 口型合成抢卡。
- 「编辑 + 导出」这一条线 4 核 / 16GB 就够。
- 真正让单机吃紧的是**并发导出 × soffice 进程数**，并发卡 1–2 即可。
- 临时文件清理要单列任务——`CosyVoice3` 那边已经吃过临时分片残留的亏。

---

## 6. 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 导出保真度不达标 | 产品核心承诺落空 | M1 先做，手工 JSON 走通再叠功能 |
| 组件 × 风格组合爆炸 | 某些组合丑/撞色 | 自动验收矩阵，M5 |
| 服务器只有 1 个中文字体 | PDF 导出难看、QA 无意义 | 给服务器装 Noto Sans CJK SC（3.4） |
| 字体在目标机器缺失 | 排版散架 | 生成端锁字体（3.4），写进产品说明；PDF 才是保真交付格式（3.5） |
| 导出与渲染抢 worker | 用户点导出没反应 | 独立导出池（3.6） |
| 用户预期「完全可编辑」 | 差评 | 文案红线（4.4） |
| Aspose 动画需求 | 商业授权成本 | 先不做，观察需求 |
