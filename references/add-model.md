# 新增 / 退役工作流模型

把新导出的 ComfyUI 工作流接进网关（REST `/v1/*` + MCP），做到「注册完就能用、不用返工」；
以及把一个档**摘下来但可复原**。

路径与环境见 `SKILL.md`。

---

## 0. 先确认工作流格式（最常见的返工点）

用户从 ComfyUI 界面「保存」出来的是 **UI 画布格式**，网关只能吃 **API 格式**：

```python
d = json.load(open(wf))
if "nodes" in d and "last_node_id" in d:  # UI 格式 → registry.load 会直接抛错
```

要求用户重新导出：`Workflow → Export (API)`，**文件名不能改**（`models.yaml` 按文件名找）。
不要自己写 UI→API 转换脚本，不如让用户重新导出可靠。

---

## 1. 分析节点，找绑定落点

打印所有节点与 inputs，标出这几类：

| 绑定 key | 找什么 |
|---|---|
| `prompt` / `negative_prompt` | 提示词编码节点（`CLIPTextEncode` 用 `text`；自定义编码节点常用 `prompt`/`negative_prompt`）。**`prompt` 是必需绑定**，缺了 registry 直接报错 |
| `image` | `LoadImage.inputs.image`（图生图/编辑模型） |
| `seed` | 采样器节点的 `seed` 或 `noise_seed`（`SamplerCustom` 是 `noise_seed`） |
| `steps` / `scheduler` / `denoise` | `BasicScheduler` 或 `KSampler` 的对应字段 |
| `cfg` | `SamplerCustom.cfg` 或 `KSampler.cfg` |
| `sampler_name` | `KSamplerSelect.sampler_name` 或 `KSampler.sampler_name` |
| `filename_prefix` | 输出节点（`SaveImage` / `SaveImageAdvanced` / `SaveVideo`） |
| `width` / `height` | 同时有 width/height 的节点（`EmptyLatentImage`） |

**判断 `output_node`**：末尾保存节点。写错会让 `collect_images` 收不到结果（报 `no_output`）。

**尺寸绑定是个关键判断**：如果工作流的 width/height 是**链接**（值形如 `["62", 0]`），说明尺寸由上游节点（如 `GetImageSize` 读输入图）决定，**不要绑**，绑了会破坏链接且 `size` 参数无效。

---

## 2. 写 models.yaml

```yaml
  <model-name>:
    workflow: <file.json>          # 与 workflows/ 下文件名一致
    description: <一句话，agent 靠它选模型，写清擅长什么>
    mode: image                     # image | video
    capabilities:
    - image-to-image                # 见下方语义说明
    output_node: '55'
    timeout: 300
    defaults:
      steps: 6
      cfg: 1
      sampler_name: euler
      scheduler: sgm_uniform
      denoise: 1
    bindings:
      filename_prefix: 55.inputs.filename_prefix
      prompt: 45:36.inputs.prompt
      seed: 45:21.inputs.noise_seed
      ...
    aliases:
    - <短名>
```

容易踩的点：

1. **`capabilities` 语义**：只写 `image-to-image` → 不传 `image` 会 400（适合专做编辑的模型）；写 `text-to-image` 也能同时收图（registry 见 `image` 绑定会自动补 `image-to-image`）。
2. **全局 `defaults.params` 会被合并进每个模型**（`{**shared_defaults, **模型 defaults}`）。默认负向含 `text, watermark` —— 对「在图上写字」类模型是反作用，必须在模型自己的 `defaults` 里写 `negative_prompt: ''` 覆盖。
3. **节点 id 含冒号**（`45:36`）照抄，yaml 里是字符串不影响。
4. **参考槽不叫 `ref_image_N` 时**：视频模型的 `references` 段默认按 `ref_images.ref_image_{i}` 这套键名去校验聚合节点上的 input。若聚合节点收的是别的名字（典型：`MiniMaxH3ImageToVideo` 的首尾帧是**关键帧**槽 `first_frame` / `last_frame`，不是参考 token），用 `image_keys` 逐槽覆盖（`videos` / `audios` 同理有 `video_keys` / `audio_keys`），列表必须与节点列表等长，否则启动即报错。

> **H3 系的现成参考**：base 四支（`minimax-h3` / `-edit` / `-lift` / `-lift-edit`）是当前最完整的一组 ——
> `references` 全套槽位、`vram_adaptive` + `vram_tiers`、以及 `attention` 档位（内部名
> `sparse_start_percent` → `159.inputs.start_percent`）都接好了。新增同族档时照抄它。

---

## 3. 本地离线校验（不必重启 ComfyUI）

```python
from gateway.registry import registry, build_workflow
registry.load(Path('models.yaml'), Path('workflows'), '', '')
spec = registry.resolve('<model-name>')
wf = build_workflow(spec, {...测试 values...}, None)
```

`registry.load` 会做 `_validate_bindings`（路径不存在直接抛错）与 `_validate_references`。再打印渲染后的关键节点，确认：
- `image` 落到 LoadImage、`prompt` 落到编码节点
- **原本是链接的字段仍是链接**（没被覆盖）

---

## 4. 热加载 + 端到端

```
curl -s -X POST http://127.0.0.1:8188/admin/reload    # 返回 reloaded:true 与模型清单
```

**先零成本验提交图，再决定是否真跑**（低显存机器尤其重要，见 §4.1）；确认无误后图生图/编辑模型走
`POST /v1/images/edits`（multipart），**必须显式传 `model`**（不传会用默认文生图模型 → 400）。

看结果图确认效果（生成完 Read 输出 png 目视检查，别只看 HTTP 200）。

### 4.1 零成本验证「实际推给 ComfyUI 的图」（不占 GPU）

只改了参数注入 / 绑定 / 拓扑时，**不要靠真跑一遍来确认**——提交后立刻取消就能拿到完整提交图。
**前提**：模型得先在这个运行实例里注册成功（`GET /v1/models` 能看到）。改过 `.py` 之后直接
`POST /admin/reload` 会返回 500 —— 旧进程不认识新 yaml 字段（如新加的 `image_keys`），
日志里的真因是 `model <name>: aggregator ... missing input ref_images.ref_image_0` 这类校验错。
**别把这个 500 当成 yaml 写错**：先在本地离线 `registry.load` 复现一遍，通过了就说明只差重启。
**顺序很重要，错一步要么白占 GPU、要么拿不到图**：

1. 提交时**必须** `background:"pending"`。漏了就走同步路径：客户端 30s 后报 timeout，
   而服务端**照跑到底**、真占 GPU 真落产物（曾因此莫名跑出一条 3.5 分钟的片子）。
2. **等 `prompt_id` 出现再动手**——它在提交响应里是异步回填的，常常是 `null`。
   轮询 `GET /roundabout/admin/queue` 按 `task.id` 取；超时再回落 ComfyUI `/queue`
   的 `queue_pending`（取最新那条）。
3. **先取快照**：`GET /roundabout/admin/queue/workflow/{prompt_id}`（或 MCP 的 `get_workflow`）。
   必须在取消**之前**取——取消后三层回落（队列 → history → 任务快照）可能全空，直接 404。
4. **再取消**：`DELETE /v1/videos/tasks/{id}` + `POST /queue {"delete":[pid]}`。
   ⚠️ **`POST /interrupt` 是全局的——只在「当前正在跑的 prompt_id 恰好是你自己那条」时才发**。
   否则会把别人（用户手动提交、另一个 agent/客户端）正在跑的活儿一起打断；被打断的那条在
   ComfyUI history 里只留一条 `execution_interrupted`、网关侧记成 `cancelled` + `code 499`，
   **从结果上根本看不出是谁打断的**。判法：先 `GET /queue` 读 `queue_running[0][1]` 与自己那条
   比对，不是自己的就**只删自己的队列项、绝不 interrupt**。
   网关任务与 ComfyUI 队列是两套，都要做，否则后端仍继续跑。
5. **最后确认队列真的空了**：`GET /queue` 的 `queue_running` / `queue_pending` 都是 0。
   打扫逻辑要**独立于**探测逻辑——写成「取到快照才取消」的话，取不到快照那次就漏了取消。
6. 逐节点 print 目标 inputs：参数落在对的节点、原本是链接的字段仍是链接。

**三个实测出来的坑（都踩过）**：

- **取消有竞态**：`DELETE /v1/videos/tasks/{id}` 返回 200 后**立刻**查 `/queue` 可能仍显示
  `queue_running=1`，等 ~0.5s 才归零。判「真取消了吗」要**同时**看网关侧任务状态（应为 `cancelled`），
  否则会误以为没清干净而重复打扫。
- **网关会把「外部路径」的参考视频/音频复制进 `input/`**（`{trace}_refvid_{N}.mp4` /
  `_refaud_{N}.wav`，`trace` = `uuid4().hex[:12]` + `-{批次号}`）；本来就在 `input/` 里的图/音频沿用原名、不复制。
  **验证完要自己清掉这些副本**。（历史命名曾带 `hermes_` 前缀，已去掉——若在旧日志/旧产物里看到，别照着找。）
- **零成本判据**：跑完后查 `output/**` 最近几分钟的新增文件数 —— **0** 才说明确实没跑到出片
  （只看 HTTP 200 或队列状态证明不了）。

**MCP 会话**：`MCP_STATELESS` 默认 `true`（无状态），所以 ComfyUI 重启后 MCP 调用不该失效；
只有显式设成 `false`（为了拿异步任务的完成推送）才会出现「客户端一直复用旧 session id、
所有调用回 `-32600 Session not found`、且不自动重新握手」——那就改走上面这套 REST。
该开关是**启动期**读的：只改 `.env` 不重启不生效；判据是向 `/mcp` POST 一个 `initialize`，
**响应头带 `mcp-session-id`** 就说明当前是有状态模式（无状态不建会话、不发这个头）。

### 4.2 真跑验收：参考通道是否真的生效（要占 GPU）

改的是参考槽（`references` / 剪枝）时，零成本快照只能证明「接上了」，证明不了「模型真的用了它」。
真跑一条（3s @1344x768），提交照 4.1 用 `background:"pending"` + 轮询：

- **提交图**：产物 mp4 内嵌的 `prompt` tag 就是提交图，逐槽核聚合节点的 `ref_images.*` /
  `ref_videos.*` / `ref_audios.*`（MiniMax H3 系聚合节点 `136`；FastH3 是 `105:145` / `105:104`，节点 id 带子图前缀）。**参考视频给两个槽**（`ref_video_N` + `ref_video_audio_N`）。
  **没装 ffprobe 时**用随包安装的 ffmpeg（`imageio_ffmpeg` 自带）：
  `<ComfyUI python 的 site-packages>\imageio_ffmpeg\binaries\ffmpeg-win-x86_64-v7.1.exe`。
  `ffmpeg -i` 读流信息（分辨率/时长/音轨）；`-f ffmetadata meta.txt` 导出全部 tags —— 但**它对 `\`
  转义**，读回来要逐字符反转义再 `json.loads`，否则报 `Invalid \escape`。
- **只给一种参考时核对剪枝**：单图 22 节点、单视频 23、单音频 22，`Load*` 节点须与传的通道一一对应。
- **视觉**：`ffmpeg -ss T -frames:v 1` 抽 3 帧肉眼过（别只看 HTTP 200）。**产物名形如
  `MiniMax_H3_00073_.mp4`——扩展名前有个下划线**，漏了 ffmpeg 直接报 no such file。
  保存前缀 `video/FastH3` 是「**子目录 + 文件名前缀**」两段：产物落在 `output/video/FastH3_00001_.mp4`，
  **不是** `output/video/FastH3/...` 目录（按后者找文件会落空）。
- **首尾帧槽专测**（`image_keys` 覆盖关键帧槽时必须做）：给两张**明显不同**的图（如「雾林狐狸」+
  「窗边橘猫」），跑完抽成品的**首帧**（`-frames:v 1 -update 1 f.png`）与**尾帧**
  （`-sseof -0.1 -i in.mp4 -frames:v 1 -update 1 l.png`）肉眼比对：首帧像图 1、尾帧像图 2，才证明
  `first_frame` / `last_frame` **两个槽都真吃进去了**。只看节点数/连线只能证明「接上了」，
  证明不了「没被静默丢弃」。
- **音频**（光听不靠谱，用客观量对照参考素材）：
  ```
  ffmpeg -i out.mp4 -af volumedetect -f null -                     # mean/max 电平
  ffmpeg -i out.mp4 -af aspectralstats=measure=centroid -f null -  # 频谱重心
  ```
  基线：参考 wav 是稳定 ~446Hz 测试音（RMS 恒 -11.73dB）→ 只给音频参考时生成音轨重心
  **485Hz** / mean **-14.1dB**；只给图时 **-44.2dB** / 3.76kHz（近静音底噪）；只给视频时跟着
  参考视频自带音轨走。**重心与电平贴近参考 = 该通道确实生效。**
  （这三个值是**通道判别用的相对关系**，不是性能预测 —— 换机器依然成立。）

---

## 5. 同步文档

- `API.md`：模型清单表、视频/编辑模型用法、必要时排障表。
- `WORKFLOWS.md`：工作流层机制（含「聚合节点不收 `ref_image_N` 时用 `image_keys`」一节）。
- `README.md`：模型总表与权重清单——**新增/下线模型会动到其中的计数与档位描述，别只改 API.md**。
- `mcp_server.py`：**仅在新增了「请求字段」时才需要动**（单纯加模型不用改）。MCP 形参逐个手写、
  与 REST 的 pydantic 模型是两条独立路径，改完必须跑 `tests/test_mcp_param_coverage.py`。

改完跑 `ROUNDABOUT_VRAM_GB=8 tests/run_tests.py`（当前基线 **20 passed / 0 failed / 2 skipped**）。

---

## 6. 退役 / 下线一个模型（与新增对称：**摘档 + 留痕**）

C 的口径往往是「先下掉吧」（先例：`minimax-h3-turbo{,-edit}` 09-20、`minimax-h3-hyperflow` 09-21）
—— **「先」= 可复原**，所以既不要真删文件、也不要在 yaml 里删干净了事，更不要把文档一次抹平。

七个落点，漏一个就是文档与实现打架：

1. **工作流 JSON 移出 `workflows/`**：`shutil.copy2` 到 `.workbuddy/stash/` 再 `unlink`。
   留心那些一次性实验脚本（`.workbuddy/` 下的 `ladder_builders.py` / `morning_run*.py` /
   `native_ab.py` / `pilot_*.py` …）——它们常把某个工作流当模板源，删了会 `FileNotFoundError`。
   响亮失败可接受，但要在日志里点名。
2. **`models.yaml` 删条目 + 原地留下线注记**：注记 = 下线日期 / 为什么跑不通（要具体到机制）/
   备份路径 / 复原步骤。别只留空行——下一个人（含未来的你）只有这一条线索。
3. **同文件顶部段落注释**：族名枚举（如「FL2VA（base / … ）」那行）里也要把它摘掉。
4. **测试**：`tests/test_save_prefix.py`（保存前缀常量 / `FAMILIES` 名单 / 目录约定注释 / 族计数）
   + `tests/test_fasth3.py`（视频档总数断言）。**计数断言是硬闸，不改必红**。
5. **文档**：`README.md` 通常有 6–8 处 —— 内置模型表（改删除线 + 写清为什么跑不通）、权重表、
   权重形态坑、社区转换版坑、一键下载脚本（注释化而非删掉）、按模型列的依赖表、第三方节点依赖行。
   `API.md` 的**可用模型表要删行** + 该档特有的参数提醒（例：hyperflow 的「传 steps 无效」）一并删。
   ⚠️ 顺手核一眼表里有没有**更早下线的档**残留（曾发现 `-turbo-edit` 还挂在 API.md 模型表里）。
6. **权重文件不删**：LoRA / diffusion_models 留在盘上（用户数据，删除不可逆），只在 README 权重表
   标「已下线档的 LoRA，无工作流引用」。
7. **记忆同步**：`MEMORY.md` 的 H3 家族段首维护一条「已下线的档（勿再注册）」汇总（含日期与原因）
   + 支数 + 待修清单；`platform-notes-full.md` 对应章节标题加【已下线】标记，正文保留（历史价值）。

**验证四道，缺一道不算收工**：

1. `ROUNDABOUT_VRAM_GB=8 tests/run_tests.py` —— 基线 **20 pass / 0 fail / 2 skip**；
2. 本地直读 registry：模型总数、`s.mode == "video"` 支数、**别名逐个 `resolve` 应全部 ModelNotFound**
   （只查主名不够 —— 别名是独立的注册面）；
3. `POST /admin/reload` 后 `/v1/models` 查无此档。⚠️ **reload 前仍是旧数**，拿它当「没生效」的判据会误判；
4. 全仓残留扫描：按关键词（`hyperflow` 之类）扫 `*.py/*.md/*.yaml/*.json`，逐行标「下线注记 / 待查」。
   允许剩下的只有注记自身与 `.cache/` 历史快照（`.workbuddy/` 被 ripgrep 默认跳过，要单独扫）。

**体量提示**：典型一次会动 7 个文件、十几处替换 + 1 个文件移出 ⇒ **一律用脚本**
（锚点 + 唯一命中断言 + 单次写盘，见 `SKILL.md` 的「执行通道」一节），手工连发 Edit 一定出乱子。
写脚本时按 `newline` 原样保留行尾，否则整个文件会因 CRLF↔LF 全量重写，diff 没法看。
