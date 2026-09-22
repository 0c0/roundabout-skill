---
name: roundabout
description: ComfyUI-Roundabout 网关（ComfyUI 自定义节点）的使用手册 —— 用 MCP 工具出图出视频、选模型与传参、异步任务与产物、注册与摘除工作流模型、下载权重、排障，以及核对这套文档与实现的说法是否一致。Use when the user 说「roundabout」「走网关生成」「generate_image」「generate_video」「MCP 生图/生视频」「/v1/images/generations」「/v1/videos/generations」「注册工作流」「models.yaml 加一个模型」「把这个档下掉」「权重下不下来」「文件名对不上」，或需要驱动装了本节点的 ComfyUI 出图出视频。
agent_created: true
---

# Roundabout 网关

把 ComfyUI 的工作流变成 **OpenAI 兼容 REST + MCP** 服务。
**不碰画布、不改代码**：在 ComfyUI 里搭好的流程 → 导出 API 格式 JSON → 在 `models.yaml` 写一段
参数映射 → 就成了一个可被任意客户端调用的 `model`。

## 怎么调：先用工具

MCP 工具是本节点**自己注册**的 —— 它们在，就说明节点已经在跑，**你手上什么都不缺**：

```
list_models → generate_image / edit_image / generate_video → get_task → get_view_url
```

**直接用，不要先探测环境**：不要找 ComfyUI 目录、不要 curl 端口、不要问地址。
工具在，说明服务在；工具不在，说明节点没跑 —— 那是部署问题，回报即可，本 skill 到不了那里。

> 显存 / 分块档位**不用管**：节点 init 时就自己探好了，并已写进模型的默认值。
> 别传分块参数，也别判断显存大小。

REST（`/v1/*`）与 MCP **完全互通**，仅当手上没有 MCP 通道时才走它 —— 见 `references/usage.md`。

## 读取路由

| 你在做什么 | 读哪个 |
|---|---|
| **调用**：选哪支模型、传什么参数、异步任务、拿产物 | `references/usage.md` |
| **接入**：把新工作流接进网关 / 摘掉某个档 | `references/add-model.md` |
| **接入 / 排障**：权重下不下来、放哪个目录、报错 `Value not in list` | `references/weights.md` |
| **改文档后**：核「文档说的和实现是否一致」 | `references/doc-audit.md` |

## 分类与生效

- **只需部署者做一次**：跑通服务、装齐权重（见 `references/weights.md`）。你调用的前提是这个已完成。
- **接入 / 排障阶段才需要的环境值**（`<ComfyUI>` 目录、`<ComfyUI python>`、项目根
  `<ComfyUI>/custom_nodes/ComfyUI-Roundabout/`）：只在要**读改仓库文件或跑脚本**时才去找；
  调用阶段一次都用不到。做法见 `references/add-model.md`。

**改完怎么生效，分两档**：

| 改了什么 | 生效方式 |
|---|---|
| `models.yaml` / `workflows/*.json` | `POST /admin/reload` 热加载 |
| `gateway/*.py` · `mcp_server.py` · `__init__.py` | **必须重启 ComfyUI** —— reload 不重载 Python |

## 三条最容易踩的机制（先记住再动手）

1. **`workflow_overrides` 是通用路径注入器**，与参数白名单**无关**：
   `{"910.inputs.scale": 2.0}` 直接改节点，路径不存在会 400。想临时掉一个旋钮不必先注册成请求字段。
2. **字段注入有三道闸（白名单 → `bindings` → `values`），缺一道就静默无效**。
   ⛔ 不在白名单的 binding 被丢掉，**只打 `unknown binding key` 警告**；
   ⛔ **`defaults` 也只对「同时出现在 `bindings` 里」的键生效**
   ⇒ 只写 `defaults` 却不配 binding ＝ **双重死键**，真实值只剩工作流模板字面值。
   ⛔ 新增 REST 字段必须同步 MCP 形参 + values 字典（守护测试 `tests/test_mcp_param_coverage.py`）。
   改完 yaml 不用猜：reload 之后翻日志，有 `unknown binding key` 就是没接上。
3. **找不到落点时先看提交图**：MCP `get_workflow`，或产物内嵌的提交图（`.png` 的 `prompt` tag /
   `.mp4` metadata）。**判「某发实际跑了什么」只看三处**：`/history` 提交图、日志的 `N/M` 行、
   产物 metadata —— 别信文件名或 tag 里的标签，标签与实参可以对不上。

**新工作流先在 ComfyUI 原生 `/prompt` 验通，再落 `models.yaml`** —— 顺序不能倒，
理由与做法见 `references/add-model.md`。

## 开工前

- 先 `list_models` 看当前实例实际注册了什么，别照文档里的名册硬写（**只看总数会被「删 2 加 2」骗过**）。
- 交付后**目视产物**，不要只看 HTTP 200 / 任务 completed。
- **不要把实测耗时 / 加速比写进文档**：换机器就不成立，写了就是待维护的错数（见 `references/doc-audit.md`）。
- 改了仓库（模型、参数、拓扑、文档）→ 跑 `tests/run_tests.py`，清点见 `references/add-model.md` §5。

## 边界

- 本 skill 只描述**节点本身**：注册表、参数注入、任务与产物、运维。
  里面出现的路径、端口、显存都**不要照抄固定值** —— skill 要能跨机器、跨机器人的 ComfyUI 用。
- **不含环境相关的测试流程、一次性脚本与本地笔记**。
- **H3 的提示词写法与能力边界不在本 skill**，见配套的 `h3-playbook`。
