---
name: roundabout
description: ComfyUI-Roundabout 网关（ComfyUI 自定义节点）的使用手册 —— 选模型、走 REST/MCP 生成、注册与摘除工作流模型、下载权重、排障与运维，以及核对这套文档与实现的说法是否一致。Use when the user 说「roundabout」「走网关生成」「/v1/images/generations」「/v1/videos/generations」「generate_video」「MCP 生图/生视频」「注册工作流」「models.yaml 加一个模型」「把这个档下掉」「权重下不下来」「文件名对不上」，或需要驱动装了本节点的 ComfyUI 出图出视频。
agent_created: true
---

# Roundabout 网关

把 ComfyUI 的工作流变成 **OpenAI 兼容 REST + MCP** 服务。
**不碰画布、不改代码**：在 ComfyUI 里搭好的流程 → 导出 API 格式 JSON → 在 `models.yaml` 写一段
参数映射 → 就成了一个可被任意客户端调用的 `model`。

## 功能边界

- **本 skill 只描述节点本身**：注册表、参数注入、任务与产物、运维。
  里面出现的路径、端口、显存都**按现场探测取**，不要照抄固定值 —— skill 要能跨机器、跨机器人的 ComfyUI 用。
- **不含任何环境相关的测试流程、一次性脚本与本地笔记**：那些是使用现场的事，不是节点的行为。
- **H3 的提示词写法与能力边界不在本 skill**，见配套的 `h3-playbook`。

## 路径与环境（这几个值不要猜，按顺序拿）

1. `<ComfyUI>`：先反推 —— 手上只要有 `custom_nodes/<节点>/` 下的任意文件，它的上两级就是 `<ComfyUI>`；
   反推不出来就问用户「ComfyUI 装在哪个目录」。
2. `<ComfyUI python>`：按序探测 `<ComfyUI>/../python/python.exe`（aki 便携包）→
   `<ComfyUI>/../python_embeded/python.exe`（官方 portable）→ `sys.executable`；都不成立就问用户。
3. 宿主与端口：默认 `8188`（ComfyUI 自身端口，REST 与 MCP 共用）。**别硬编内网地址** ——
   先 `GET http://127.0.0.1:8188/health` 探，探不到再问用户「平时怎么访问 ComfyUI」。

- 项目根：`<ComfyUI>/custom_nodes/ComfyUI-Roundabout/`
- REST 端点 `/v1/*` 与 MCP 端点 `/mcp` **都挂在 ComfyUI 端口**；MCP 可注册进任意 MCP 客户端。
- 探到的值可以就近记下来，但**不要写回本 skill**。

## 读取路由

| 什么时候 | 读哪个 |
|---|---|
| 要调用（选哪支模型、传什么参数、异步任务、拿产物、运维） | `references/usage.md` |
| 要把新工作流接进网关 / 摘掉某个档 | `references/add-model.md` |
| 权重下不下来、该下哪些、放哪个目录、文件名对不上、报错说 `Value not in list` | `references/weights.md` |
| 要核「文档说的和实现是否一致」 | `references/doc-audit.md` |

## 三条最容易踩的机制（先记住再动手）

1. **`workflow_overrides` 是通用路径注入器**，与 `KNOWN_PARAMS` 白名单**无关**：
   `{"910.inputs.scale": 2.0}` 直接改节点，路径不存在会 400。想临时掉一个旋钮不必先注册成请求字段。
2. **字段注入有三道闸（`KNOWN_PARAMS` → `bindings` → `values`），缺一道就静默无效**。
   ⛔ `_normalize_bindings` 把不在白名单的 binding 丢掉，**只打 `unknown binding key` 警告**；
   ⛔ **`defaults` 也只对「同时出现在 `bindings` 里」的键生效**
   （`build_workflow`：`effective = {k: v for k, v in spec.defaults.items() if k in spec.bindings}`）
   ⇒ 只写 `defaults` 却不配 binding（或配了被上面丢掉的）＝ **双重死键**，真实值只剩工作流模板字面值。
   ⛔ `values` 只从 `pipeline._video_values` 的**显式字段表**构造，请求体的额外字段（`extra="allow"`）
   到不了那里 ⇒ 新增 REST 字段必须同步 MCP 形参 + values 字典，守护测试 `tests/test_mcp_param_coverage.py`。
   改完 yaml 不用猜：`/admin/reload` 之后翻日志，有 `unknown binding key` 就是没接上。
3. **找不到落点时先看提交图**：`GET /roundabout/admin/queue/workflow/{prompt_id}`（或 MCP `get_workflow`），
   或产物内嵌的提交图（`.png` 的 `prompt` tag / `.mp4` metadata）。
   **判「某发实际跑了什么」只看三处**：`/history` 提交图、日志的 `N/M` 行、产物 metadata ——
   别信文件名或 tag 里的标签，标签与实参可以对不上。

**生效方式分两档**：`models.yaml` / `workflows/*.json` 走 `POST /admin/reload`（热加载）；
`gateway/*.py` · `mcp_server.py` · `__init__.py` **必须重启 ComfyUI** —— `/admin/reload` 不重载 Python。

## 增改工作流：先在 ComfyUI 原生接口验，再进网关

**顺序不能倒**：新工作流先用 **ComfyUI 自己的 `/prompt`** 跑通 —— API 格式 JSON 直接 POST，
`/history/{prompt_id}` 取结果 —— 确认节点接线、权重文件、分辨率都对，**验过之后再落 `models.yaml`**
交给 roundabout。理由是网关在中间叠了 references 接线、bindings 注入、任务表与产物缓存：
出问题时你分不清是「工作流本身错」还是「网关那几层错」，而且排查一样要占 GPU。

⚠️ **别用耗时推断设备**：`ImageUpscaleWithModelBatched` 这类节点走 `get_torch_device()`，
看着慢不等于回落 CPU。要判定就在跑的同时采
`nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv,noheader`（GPU 会 ~100% 且显存抬升）。

## 常用命令

```
# 仓库自带测试（默认离线，不占 GPU）
<ComfyUI python> tests/run_tests.py         # 冒烟
<ComfyUI python> tests/run_tests.py --all   # 含会真跑 ComfyUI 的用例
# ROUNDABOUT_VRAM_GB=<GiB> 可钉住显存档位（不填则自动探测）

# 生图 / 生视频
curl -X POST http://127.0.0.1:8188/v1/images/generations -H "Content-Type: application/json" \
     -d '{"model":"z-image-turbo","prompt":"a red fox in snow","size":"1024x1024"}'

# 热加载（只对新 yaml / 新工作流有效）
curl -X POST http://127.0.0.1:8188/admin/reload

# 当前状态
curl http://127.0.0.1:8188/health                    # 含模型清单 —— 只看总数会被「删 2 加 2」骗过
curl http://127.0.0.1:8188/roundabout/admin/queue    # 队列
curl http://127.0.0.1:8188/roundabout/admin/weights  # 权重体检（缺哪些 + 下载命令）
```

## 提交前的三条自检

- 改了模型/参数 → 跑 `tests/run_tests.py`（新增请求字段还要同时跑 `test_mcp_param_coverage.py`）。
- 改了文档 → 核一遍**计数类**（模型支数 / 工作流文件数 / 权重文件数 / MCP 工具数）是不是脚本枚举出来的，
  别沿用旧数。**不要把实测耗时写进文档**（见 `references/doc-audit.md`）。
- 改了工作流拓扑 → 同时核 `models.yaml` 的 bindings 与文档里出现的**节点 id**。
