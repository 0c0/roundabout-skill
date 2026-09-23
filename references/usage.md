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
| `GET /roundabout/view` | 可视化页面（浏览 input/output + 实时进度） |

管理端点（`/roundabout/admin/*`，工作流上传/删除、`models.yaml` 读写与结构化编辑）见仓库 `API.md`。

## 3. MCP 工具（默认开启）

`get_tool_info` · `generate_image` · `edit_image` · `remove_background` · `generate_video` ·
`list_models` · `get_task` · `cancel_task` · `queue_status` · `get_workflow` ·
`reload` · `health` · `get_view_url` · `get_skills` · `check_weights`

传输 `streamable-http`，端点 `/mcp`（与 ComfyUI 同端口），与 REST 完全互通。
实际工具数以 `tools/list` 为准。

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
> ① `tests/test_mcp_default_on.py` 的 `tools/list` 计数断言（自守）；
> ② `toolinfo._endpoints()['mcp']`，与 `@mcp.tool` 注册集做 AST 比对（`tests/test_toolinfo.py`）；
> ③ `README.md` 的 MCP 工具清单 + `API.md` 的架构图计数 / §7 标题 / 每行工具表 —— 同测试的
>    「文档里的工具清单不漂移」区块，**新增工具忘改文档、或文档写了不存在的工具都会直接红**；
> ④ **本文件的工具清单，只有这一处靠人记。**

## 4. 关键请求参数

| 参数 | 说明 |
|---|---|
| `model` | 默认 `z-image-turbo`（`DEFAULT_MODEL` 可覆盖） |
| `size` | 视频：`<tier>p-<ratio>` 或 `<ratio>@<tier>`，tier ∈ {`480p`,`576p`,`720p`,`768p`,`1080p`,`1440p`}，ratio ∈ {`1:1`,`3:4`,`4:3`,`16:9`,`9:16`}；或直接 `WxH`。**1440p 在 8 GiB 档直接生跑不动，要更大画面走 lift** |
| `duration` | 秒，网关允许 1–15。⚠️ H3 系权重的训练区间是 **124–362 帧 ≈ 5–15 s**，`d≤4` 落在分布外 —— **别拿 d≤4 的产物下画质结论**（快速跑通用可以） |
| `attention` | `sparse`（默认，块稀疏加速，更快更省显存）/ `dense`（关闭稀疏换致密画质，更慢更吃显存）。**只对 base 四支开放**（`minimax-h3` / `-edit` / `-lift` / `-lift-edit`）；FastH3 两支恒定稀疏，传了报 400 |
| `reference_images` / `_videos` / `_audios` | 参考素材；按请求实际提供数量裁剪，未传的槽提交前从图里删掉。**图像档也吃 `reference_images`**（`qwen-image-2.1` 6 槽 / `flux2-klein-image-edit-turbo` 4 槽）；`image` 只收单张，多图必须走这里，超上限报 400。`qwen-image-2.1` 是**文生与多图编辑同一支**：一张参考都不传即纯文生（这时 `size` 才生效），1–6 张则输出尺寸跟随第 1 张参考图 |
| `background:"pending"` | 异步；立即返回 task，用 `get_task` / `GET /v1/videos/tasks/{id}` 轮询 |
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
