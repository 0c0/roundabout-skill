---
name: roundabout
description: ComfyUI-Roundabout 网关的总入口 —— 选模型、走 REST/MCP 生成、注册或下线工作流、下载权重、排障运维，以及核对这套文档与实现的说法是否一致。Use when the user 说「roundabout」「走网关生成」「/v1/images/generations」「/v1/videos/generations」「generate_video」「MCP 生图/生视频」「注册工作流」「models.yaml 加一个模型」「把这个档下掉」「权重下不下来」「文件名对不上」，或需要驱动本地 ComfyUI 出图出视频、以及审查这仓库的介绍类文档是否过期。提示词与能力边界转 skill `h3-official-playbook`，注意力档位转 `h3-sparse-attention`，H3 放大转 `selflift-progressive-upscale`，Z-Image 中文提示词转 `zimage-prompt-writing`，子图工作流 JSON 改写转 `comfyui-subgraph-workflow-edit`。
agent_created: true
---

# Roundabout 网关

把本机 ComfyUI 的工作流变成 **OpenAI 兼容 REST + MCP** 服务。
**不碰画布、不改代码**：在 ComfyUI 里搭好的流程 → 导出 API 格式 JSON → 在 `models.yaml` 写一段
参数映射 → 就成了一个可被任意客户端调用的 `model`。

- **路径约定**：`<ComfyUI>` = ComfyUI 安装目录（形如 `X:/.../ComfyUI`）。
- **这几个值不要猜，按顺序拿**：
  1. `<ComfyUI>`：先反推 —— 手上只要有 `custom_nodes/<节点>/` 下的任意文件，它的上两级就是 `<ComfyUI>`；反推不出来就问用户「ComfyUI 装在哪个目录」。
  2. `<ComfyUI python>`：按序探测 `<ComfyUI>/../python/python.exe`（aki 便携包）→ `<ComfyUI>/../python_embeded/python.exe`（官方 portable）→ `sys.executable`；都不成立就问用户。
  3. 宿主与端口：默认 `8188`（ComfyUI 自身端口，REST 与 MCP 共用）。**别硬编内网地址** —— 先 `GET http://127.0.0.1:8188/health` 探，探不到再问用户「平时怎么访问 ComfyUI」。
  探到的值可以就近记下来，但**不要写回本 skill**（skill 要能跨机器用）。
- 项目根：`<ComfyUI>/custom_nodes/ComfyUI-Roundabout/`
- ComfyUI python（跑测试 / 带 aiohttp 的脚本）：用安装包自带的解释器（下文记作 `<ComfyUI python>`，取法见上）
- 端点：REST `/v1/*` 与 MCP `/mcp` 都挂在 **ComfyUI 端口**（默认 8188）
- 对外 MCP 还有一处宿主挂载：`http://<宿主地址>:8188/mcp`（注册名 `comfyui-roundabout`）

## 读取路由

| 什么时候 | 读哪个 |
|---|---|
| 要调用（选哪支模型、传什么参数、异步任务、拿产物、运维） | `references/usage.md` |
| 要把新工作流接进网关 / 下线某个档 | `references/add-model.md` |
| 权重下不下来、该下哪些、放哪个目录、文件名对不上 | `references/weights.md` |
| 要核「文档说的和实现是否一致」 | `references/doc-audit.md` |

## 相关 skill（按任务转交，别在这里重造）

本 skill 只管**网关这一侧**（注册表、参数注入、任务与产物、运维）。下面这些各有专责：

| 任务 | 转给 |
|---|---|
| 写 / 改 H3 提示词、判 H3 能不能做、选时长与宽高比 | `h3-official-playbook` |
| 选 / 挂 / 验证 H3 的注意力后端与稀疏档（含 `start_percent` 怎么调） | `h3-sparse-attention` |
| 给 H3 加 SelfLift 渐进放大、调 σ 落点 | `selflift-progressive-upscale` |
| 写 Z-Image 中文提示词 | `zimage-prompt-writing` |
| 改子图结构的工作流 JSON（增删参考图槽、扁平化成 API 格式） | `comfyui-subgraph-workflow-edit` |
| 审 skill 自身的文档一致性 | `skill-doc-audit` |

## 执行通道（本机硬约束）

- ⛔ **Bash 工具完全不可用**（`dirname` / `ls` / `cat` 全 command not found），`cmd /c` 也不行。
  **一律用 Python 当执行通道**：`& "<ComfyUI python>" <脚本.py>`。
- ⛔ **PowerShell 会吞掉原生命令的 stderr**（`git push` 那次就是这样：rc=128、输出为空）。
  要真实报错就在 Python 里用 `subprocess.run(..., capture_output=True, text=True)` 收双流。
- **同一文件的多处改动禁止并行 Edit** —— 会互相覆盖且**零报错**。
  一律写「锚点定位 + 唯一命中断言 + 每文件单次写盘」的脚本（`text.count(old)` 必须等于预期次数，
  不符就中止不落盘）。读文件时记录 `newline` 原样写回，否则整个文件会因 CRLF↔LF 全量重写。
- **改 `.py` 必须重启 ComfyUI**；`/admin/reload` 只重读 `models.yaml` + `workflows/`。重启命令见 `references/usage.md §6`。

## 三条最容易踩的机制（先记住再动手）

1. **`workflow_overrides` 是通用路径注入器**，与 `KNOWN_PARAMS` 白名单**无关**：
   `{"910.inputs.scale": 2.0}` 直接改节点，路径不存在会 400。想临时掉一个旋钮不必先注册成请求字段。
2. **MCP 形参手写、未声明字段被静默丢弃**（`extra="ignore"`，不报错）。新增 REST 字段必须同步
   MCP 形参 + `gateway/pipeline.py` 的 values 字典，否则「看起来支持、实际落回默认」。
   守护测试 `tests/test_mcp_param_coverage.py`。
3. **找不到落点时先看提交图**：`GET /roundabout/admin/queue/workflow/{prompt_id}`（或 MCP `get_workflow`），
   或产物内嵌的提交图（`.png` 的 `prompt` tag / `.mp4` metadata）。
   **判「某发实际跑了什么」只看三处**：`/history` 提交图、`logs/comfyui_*.log` 的 `N/M` 行、产物 metadata ——
   别信 tag 名（历史上有过 `F49`/`T49` 这类命名骗过人的先例）。

## 常用命令

```
# 冒烟（离线，不占 GPU）
ROUNDABOUT_VRAM_GB=8 <ComfyUI python> tests/run_tests.py         # 基线 20 passed / 0 failed / 2 skipped
<ComfyUI python> tests/run_tests.py --all                        # 含会真跑 ComfyUI 的用例

# 生图 / 生视频（同步）
curl -X POST http://127.0.0.1:8188/v1/images/generations -H "Content-Type: application/json" \
     -d '{"model":"z-image-turbo","prompt":"a red fox in snow","size":"1024x1024"}'

# 热加载（只对新 yaml / 新工作流有效）
curl -X POST http://127.0.0.1:8188/admin/reload

# 当前状态
curl http://127.0.0.1:8188/health            # 含模型清单 —— 只看总数会被「删 2 加 2」骗过
curl http://127.0.0.1:8188/roundabout/admin/queue
```

## 提交前的三条自检

- 改了模型/参数 → 跑 `tests/run_tests.py`（新增请求字段还要同时跑 `test_mcp_param_coverage.py`）。
- 改了文档 → 核一遍**计数类**（模型支数 / 工作流文件数 / 权重文件数 / MCP 工具数）是不是脚本枚举出来的，
  别沿用记忆里的旧数。**不要把实测耗时写进文档**（见 `references/doc-audit.md`）。
- 改了工作流拓扑 → 同时核 `models.yaml` 的 bindings 与三份文档里出现的**节点 id**。
