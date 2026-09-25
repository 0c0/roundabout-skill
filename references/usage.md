# 调用面：端点、选型、参数、运维

## 1. 模型选型

**权威名册在 `models.yaml`**（`GET /v1/models` 看当前实例的实际注册结果）。按用途挑：

| 用途 | 首选 | 备选 / 备注 |
|---|---|---|
| 文生图（默认） | `z-image-turbo`（8 步） | `z-image`（30 步，更精细） |
| 文生图（Boogu 系） | `boogu-image-turbo` | `boogu-image-base`（30 步）/ `boogu-image-base-4step` |
| **改图内文字** | `boogu-image-edit-turbo` | `boogu-image-edit`（30 步） |
| **语义改写 / 换背景材质** | `flux2-klein-image-edit-turbo` | 尺寸跟随输入图，`size` 不生效，输出约 1MP |
| 抠图 | `utility-birefnet-remove-background` | promptless，走独立端点 |
| 视频（最多能力） | `minimax-h3`（FL2VA） | 文生 / 首帧 / 首尾帧 |
| 视频（参考素材） | `minimax-h3-edit`（Ref2VA） | 多图 + 多视频 + 多音频 |
| 视频（要更大画面） | `minimax-h3-lift` / `-lift-edit` | 原生采样 → 确定性 latent lift，输出画布 ×scale |
| 视频（草稿 / 快周转） | `fasth3` | 8 步蒸馏档；`fasth3-edit` 是占位档（官方未蒸馏 Ref2VA） |

**选型要诀**：要最大质量用 `minimax-h3` 系；`fasth3` 是**草稿档** —— 官方口径 8 步最优、改步数掉质量，
**不是 49/50 步的无损替代**；要分辨率走 **lift**，不是直接拉 `size`。

## 2. REST 端点

| 端点 | 用途 |
|---|---|
| `POST /v1/images/generations` | 文生图 |
| `POST /v1/images/edits` | 图生图 / 编辑（multipart，**必须显式传 `model`**） |
| `POST /v1/images/remove-background` | 抠图（promptless） |
| `POST /v1/videos/generations` | 文生 / 参考生视频 |
| `GET /v1/videos/tasks/{id}` · `GET /v1/images/tasks/{id}` | 异步任务查询 |
| `DELETE /v1/videos/tasks/{id}` · `DELETE /v1/images/tasks/{id}` | 取消任务 |
| `GET /v1/models` · `GET /v1/models/{id}` | 模型清单 |
| `GET /v1/images/files/{name}` · `GET /v1/videos/files/{name}` | 取产物（受 `OUTPUT_TTL`） |
| `GET /health` | 网关 + ComfyUI 后端健康 |
| `POST /admin/reload` | 热加载 `models.yaml` + `workflows/` |
| `GET /roundabout/admin/queue` | 队列监控合并视图（每条含 seed） |
| `GET /roundabout/admin/queue/workflow/{prompt_id}` | 取提交图快照（队列 → history → 任务快照） |
| `GET /roundabout/admin/weights` | 权重体检：内置工作流当前缺哪些权重 + 每条的下载命令 |
| `GET /roundabout/admin/tool-info` | **调用结构自描述**：逐模型「每个字段是否生效 / 区间 / 枚举 / 默认值」+ 参考槽数量 + 全局限制。加 `?view=compact` 取裁剪版、`&model=<名>` 限定单模型 |
| `GET /roundabout/view` | 可视化页面（浏览 input/output + 实时进度 + 任务看板） |
| `GET\|POST\|DELETE /roundabout/view/board[...]` | 任务看板：当前看板 / 钉入 / 删单条 / 清空归档 / 历史列表 / 归档详情 / 载回 / 删归档。完整八条见仓库 `API.md` §7.1 |
| `POST /roundabout/view/reveal` | 把某张**外部卡**指向的路径交给系统文件管理器打开（目录进入 / 文件定位）—— 仅本机访问时执行 |

管理端点（`/roundabout/admin/*`，工作流上传/删除、`models.yaml` 读写与结构化编辑）见仓库 `API.md`。

## 3. MCP 工具（默认开启）

`get_tool_info` · `generate_image` · `edit_image` · `remove_background` · `generate_video` ·
`list_models` · `get_task` · `cancel_task` · `queue_status` · `get_workflow` ·
`reload` · `health` · `get_view_url` · `get_skills` · `check_weights` ·
`view_board`

传输 `streamable-http`，端点 `/mcp`（与 ComfyUI 同端口），与 REST 完全互通。
实际工具数以 `tools/list` 为准。

**skill 版本感知（2026-09-24 起）**：`get_skills` 每个条目带 `skill_version`——服务器期望的
skill 内容版本，与各 skill 仓库 SKILL.md frontmatter 的 `skill_version` 同步 bump。
本地已装副本 frontmatter 里的版本**低于**该值 ⇒ 副本过期，按 `install_url` 重装拿新版
（server 侧调用面细节永远以 `get_tool_info` 运行期推导为准，不存在过期问题）。

**错误形态（2026-09-24 起）**：「目标不存在」一族（`task_not_found` / `prompt_not_in_queue` /
`archive_not_found` / `board_item_not_found`，含钉卡时给未知 `task_id`、删卡时给未知卡 id）**不抛工具异常**，统一回
`{ok:false, error, code, status:404}` —— 按同一结构解析即可。参数非法与上游/内部错误照常抛
（那些是 bug，不伪装成业务失败）。同步生成回执也带 `task_id`（REST `ImageResponse`/`VideoResponse`
新字段），交给 `view_board(action="pin")` 钉卡或 `get_task` 查询都行。

⚠️ **工具 `description` 只保留一句话定位**（2026-09-23 起）。参数细节 —— 逐模型生效性、区间、
枚举、默认值、参考槽数量、尺寸档位 —— **一律查 `get_tool_info`**（或 REST
`GET /roundabout/admin/tool-info`）。两份都由 `gateway/toolinfo.py` 从**运行期状态**推导，
不写散文：模型加绑定 / 改字段约束 / 换参考槽拓扑都会自动反映，因此不会像描述那样漂移
（历史教训：MCP 的 `edit_image` 描述长期挂着「编辑模型可以不传 image」这句错口径）。

> ⛔ **MCP 形参是逐个手写的，与 REST 的 pydantic 模型没有任何同步机制**；形参里没声明的字段
> 到不了工具函数，**静默丢弃、不报错**。REST 侧 pydantic 是 `extra="allow"`，额外字段能收进请求体，
> 但 `values` 只从显式字段表构造 ⇒ **它们同样进不了工作流**。所以「文档里有、传了却没生效」先怀疑这里。
> 覆盖度由 `tests/test_mcp_param_coverage.py` 守护 —— 新增 REST 字段必须在 MCP 同步，否则测试红。
> 反过来，MCP 没暴露的字段仍可走 REST 端点传（两条路径最终汇入同一个 pipeline）。
> 增删 MCP 工具要同步 **四处**：
> ① **测试里的工具计数是两处硬编码，都要数**：`tests/test_mcp_default_on.py` 的 `tools/list`
>    计数断言、`tests/test_toolinfo.py` 的「MCP 工具数为 N」—— 只改一处会漏
>    （2026-09-24 加 `get_view_board` 时先漏了后者）；
> ② `toolinfo._endpoints()['mcp']`，与 `@mcp.tool` 注册集做 AST 比对（`tests/test_toolinfo.py`）；
> ③ `README.md` 的 MCP 工具清单 + `API.md` 的架构图计数 / §7 标题 / 每行工具表 —— 同测试的
>    「文档里的工具清单不漂移」区块，**新增工具忘改文档、或文档写了不存在的工具都会直接红**；
> ④ **本文件的工具清单，只有这一处靠人记。**

### 3.1 任务看板（`view_board` 单入口，`action` 分发）

> 2026-09-24 起看板 5 个工具（pin/clear/get/history/load）合并为 **`view_board(action=…)`**，
> 并新增 `remove`（删单张卡）—— 之前 MCP 没有「删一张」的口，agent 只能清空重钉。

页面里的 **`看板` 标签页**（与 `Output` / `Input` 并列，标签上带卡片数）有一块**无限画布**。产出超过一两件时，**钉到看板上**比逐个把文件地址给用户清楚得多 ——
按语义排布好，用户一眼看全，也方便他就着这张图做总结。

- **`action="pin"`**：钉卡。产物来源 `url` / `path` / `task_id` **三选一**（优先级依次降低）；
  `task_id` 可以是异步任务的，也可以是**同步生成回执里回的那个**（图像同步链路同样留任务记录）。
  **表达排布优先用批量 `items` 数组的顺序**（分镜按序横排、A/B 两列），自动排布**永不遮挡**已有卡；
  确需精确位置才给 `x`/`y`，压到已有卡时回执带 `covered`/`warning`（后钉的在上层）。
  `w`/`h` **单位是 px 不是倍率**（默认 200×168，范围 40–2000，超界压回且回 `size_adjusted:true`），
  一般保持默认。
  没有产物也能钉 —— 传 `note` 钉纯文本卡，用来写「这轮做到哪了」；
  `path` 指向 `input`/`output` 里的**目录**时自动落成**目录卡**，用户点一下就直接进那个目录
  （「成品都在这几个目录里」也值得钉一张）；
  `path`（绝对路径）指向这两个目录**之外**、本机真实存在的目录或文件时落成**外部卡**，
  用户点它会在系统文件管理器里打开 / 定位 —— 页面上看不到内容，见本节末尾。
- **一次钉多张就传 `items` 数组**（每项同上那套字段）：一次往返、一次落盘，且严格按数组
  顺序排布 —— 逐张调不只是慢，并发时位置还会乱。批量时顶层单卡字段一个都别传
  （同传会被明确拒绝，而不是替你猜哪边生效）。
- **`action="remove"`**：删**一张**卡（`id` 必填，来自 pin/get 回执）—— 钉错一张删一张就行，
  **别清整板重钉**。
- **`action="get"`**：读**当前看板**上已有哪些卡片（标题 / 类别 / 坐标 / 说明）与归档份数，
  并**顺带回最近 3 份归档的摘要**（带 `kinds` / `preview` / `models` 指纹 ——「上一轮钉了什么」
  顺手就看到了，不必为一个问题调两件工具）。
  **要接着往上钉之前先看一眼** —— 免得钉重、或与已有卡片叠在一起；用户问「板上有什么」时也用它。
  看的是当前板。**要续接某一轮，先问用户要归档 ID** —— 页面「历史」里每份都有「复制 ID」，
  他一眼认得出是哪一轮，你列一遍摘要再猜是白花几次往返。
- **`action="clear"`**：**换任务或交付完就清空**，别把上一轮的卡片留在旁边造成混淆。
  清空**即归档**，不会丢；`label` 起个像样的名字（如「第 1 轮 · 分镜草图」）回看时好认 ——
  不给也会按内容自动命名（如「7 张 · image/text · 10:24」），不会是一排认不出的「未命名」。
- **`action="history"`**：回看某轮钉过什么；给用户做总结时用它还原「当时都出了哪些东西」。
  摘要带 `kinds` / `preview`（前 3 张标题）/ `models` 三样指纹，扫一眼就认得出该载哪一份。
  **已经拿到 `archive_id`（用户粘贴的、或上次调用的回执）就别调它** —— 要续接直接载。
- **`action="load"`**：把某份归档**载回**当前看板（**替换**语义）—— **新会话要续接上一轮任务时用**。
  `archive_id` 的**首选来源是用户**：页面历史里每份都有「复制 ID」，粘过来是一串 12 位十六进制
  （常配一句「接着这个」）—— 那就直接载入，**别再去扫历史列表**；没人给你 ID 时才用
  `action="history"` 挑一份，或干脆**留空 = 载回最近一份**。上一轮钉过的卡片会回到画布上；
  当前看板非空会**先自动归档**它（回 `auto_archived`），不会静默丢内容。
  用户自己在页面上只能**只读回看**归档 + 复制 ID（回看时还能点「用这份替换当前看板」），
  所以**别指望用户把板恢复回来** —— 需要续接就由你载。

> **要不要把页面地址给用户，由你自己判断**（`get_view_url`）：用户在等结果、这一轮钉了很多张、
> 或他明显没在看页面时，主动给（或直接替他打开）；他正盯着页面、或只是补一张图，就不必打扰 ——
> **别变成每生成一次就复读一遍地址的噪音**。

看板容量 60 张、历史保留 20 份。产物落在 `input`/`output` 之外时会标成**外部卡**：
页面上读不到内容（`pin_view_item` 会回一条 `warning` 说明），用户点它会在跑 ComfyUI 那台机器上
拉起文件管理器 —— 但**只有他本机打开页面时才有效**，远程访问会收到明确拒绝。
所以想让用户**在页面里**看到内容，产物还是得放进 `input`/`output`。

> 卡片点了什么反应，取决于**产物在哪**：图片/视频/音频弹层预览，目录卡跳进该目录，
> 外部卡拉起系统文件管理器（先确认，仅本机访问有效），纯文本卡只给一条提示。
> **回看历史归档时整块看板是只读的**（卡片 `×` 与「清空并归档」都不出现），
> 想改就先把某份用 `load_view_board` 载回当前看板。

## 4. 关键请求参数

| 参数 | 说明 |
|---|---|
| `model` | 默认 `z-image-turbo`（`DEFAULT_MODEL` 可覆盖） |
| `size` | 视频：`<tier>p-<ratio>` 或 `<ratio>@<tier>`，tier ∈ {`480p`,`576p`,`720p`,`768p`,`1080p`,`1440p`}，ratio ∈ {`1:1`,`3:4`,`4:3`,`16:9`,`9:16`}；或直接 `WxH`。**所有视频尺寸对齐 32 的倍数**（latent 偶数 × 16 下采样；`720p` 实际 736、`768p-16:9` = 1376×768，`WxH` 自动 round），实际输出看响应 `size` 回显。**默认画布 1344×768 是 7:4**，别当成 `768p-16:9`。**1440p 在 8 GiB 档直接生跑不动，要更大画面走 lift**（`output_size` 给期望输出、网关反推倍率；仅 lift 两支） |
| `duration` | 秒，网关允许 1–15。⚠️ H3 系权重的训练区间是 **124–362 帧 ≈ 5–15 s**，`d≤4` 落在分布外 —— **别拿 d≤4 的产物下画质结论**（快速跑通用可以） |
| `attention` | `sparse`（默认，块稀疏加速，更快更省显存）/ `dense`（关闭稀疏换致密画质，更慢更吃显存）。**只对 base 四支开放**（`minimax-h3` / `-edit` / `-lift` / `-lift-edit`）；FastH3 两支恒定稀疏，传了报 400 |
| `first_frame` / `last_frame` | **首尾帧**（`minimax-h3` / `-lift` / `fasth3`）：帧会实际成为输出的第一/最后一帧，按画布 size cover 裁剪（等比铺满 + 居中裁，不变形）；单图也可用 `first_frame` 只给首帧。**可与 `reference_images` 同传**（v1.17 统一节点拓扑：帧槽=帧语义、参考槽=conditioning 语义）；无帧槽的模型传了 400 |
| `reference_images` / `_videos` / `_audios` | 参考素材（H3 六支 + 图像档；**帧与参考图可同传**）。参考视频/音频=视频编辑 / 动作 / 运镜 / 音频复用，主口径走 Ref2VA 权重（`minimax-h3-edit` / `-lift-edit`）—— FL2VA 权重（`minimax-h3` / `-lift`）对视频参考的消费**未标定**，音频参考 09-25 实测**不迁移音色**（输出 H3S 自有音色，音色/台词复用走 edit 档）；按请求实际提供数量裁剪，未传的槽提交前从图里删掉。**图像档也吃 `reference_images`**（`qwen-image-2.1` 6 槽 / `flux2-klein-image-edit-turbo` 4 槽）；`image` 只收单张，多图必须走这里，超上限报 400。`qwen-image-2.1` 是**文生与多图编辑同一支**：一张参考都不传即纯文生（这时 `size` 才生效），1–6 张则输出尺寸跟随第 1 张参考图。edit 档输出尺寸由 `size` 决定、与参考图无关 |
| `background:"pending"` | 异步；立即返回 task，用 `get_task` / `GET /v1/videos/tasks/{id}` 轮询。**轮询到 `completed` 时回执顶层就有 `size`**（实际输出尺寸，与同步响应同位），不必再 ffprobe 产物 |
| `response_format` | `url`（默认）/ `path`（落盘绝对路径）/ `b64_json` |
| `filename_prefix` | 落盘前缀，可含 `/` 建子目录 |
| `workflow_overrides` | **通用路径注入器**：`{"910.inputs.scale": 2.0}`。REST 与 MCP 都暴露，`set_path` 只要求「最后一跳命中已存在的键」，**与 `KNOWN_PARAMS` 白名单无关**。路径不存在会报 400（typo 会炸出来，不会静默无效） |

**同步 vs 异步**：图片请求始终同步返回；视频建议 `background:"pending"` —— 同步路径下客户端容易
先超时，而服务端**照跑到底**（真占 GPU 真落产物）。

## 5. 运维

| 改了什么 | 生效方式 |
|---|---|
| `models.yaml` / `workflows/*.json` | `POST /admin/reload`（或面板「重新加载」） |
| `gateway/*.py` · `mcp_server.py` · `__init__.py` | **必须重启 ComfyUI** —— `/admin/reload` 不重载 Python |
| `web/*.html` / `*.js` | 浏览器强刷 |

常用环境变量（完整清单见仓库 `API.md`）：

| 变量 | 默认 | 说明 |
|---|---|---|
| `MCP_ENABLED` | `true` | 嵌入式 MCP 网关 |
| `MCP_STATELESS` | `true` | 无状态 MCP：ComfyUI 随便重启，agent 不用重连。设 `false` 才有「完成推送」，代价是重启后旧会话全废 |
| `DEFAULT_MODEL` | YAML 内值 | 默认模型（优先级高于 YAML） |
| `MAX_CONCURRENCY` | `2` | 并发生成上限 |
| `JOB_TIMEOUT` / `OUTPUT_TTL` | `300` / `3600` | 任务超时（秒，`0`=不限、仅由 ComfyUI 状态判定）/ 产物 url 存活期 |
| `PUBLIC_BASE_URL` | 空 | 产物 url 绝对化基准，**远程部署必填** |
| `OPENAI_GATEWAY_API_KEYS` | 空 | 填了才启用鉴权（逗号分隔） |
| `ROUNDABOUT_VRAM_GB` | 空 | 手动钉住显存档位（GiB）；不填则自动探测，探测不到就不覆盖工作流自带值 |

**`MCP_STATELESS` 是启动期读的**：只改 `.env` 不重启不生效；判据是向 `/mcp` POST 一个 `initialize`，
**响应头带 `mcp-session-id`** 就说明当前是有状态模式（无状态不建会话、不发这个头）。
有状态模式下 ComfyUI 重启会让客户端复用旧 session id、所有调用回 `-32600 Session not found`
且不自动重新握手 —— 那就改走 REST。

**显存自适应（`vram_adaptive`）**：模型声明后，网关**启动时**探测显存，从 `defaults.vram_tiers`
取「`min_gb` 不超过实际显存」的最大一档作为分块参数默认值。规则：档位值 < 模型 `defaults` < 请求参数。
`min_gb: 0` 是兜底档（更小的卡不会「无档可匹配」，但最低两档拿到的是**同一套**参数）；
超过最高档也全部落到最高档。
⛔ **没有 OOM 回退机制** —— 分块仍 OOM 就 `unload_all_models()` → 直接 FAILURE，不降档不重试。

## 6. 重启命令

```
<ComfyUI python> <ComfyUI>/main.py
    --auto-launch --preview-method auto --cuda-malloc --use-ck-attention
```

无 `--port` → 8188。找在跑的进程（Windows）：`Get-CimInstance Win32_Process` 筛
`CommandLine -like '*main.py*'`。重启后回读 `GET /health` 的 `models` 数组验收
（**只看总数会被「删 2 加 2」骗过**）。
