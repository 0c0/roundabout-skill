# roundabout-skill

ComfyUI-Roundabout 网关（ComfyUI 自定义节点）的使用手册 skill —— 把 ComfyUI 工作流变成 **OpenAI 兼容 REST + MCP** 服务，**不碰画布、不改代码**：搭好流程 → 导出 API 格式 JSON → 在 `models.yaml` 写参数映射 → 即可被任意客户端调用。

## 怎么调

MCP 工具由节点自己注册——工具在，说明节点在跑，直接用：

```
list_models → generate_image / edit_image / generate_video → get_task → get_view_url
```

**不要先探测环境**（不找目录、不 curl 端口、不问地址）。REST（`/v1/*`）与 MCP 完全互通，仅当没有 MCP 通道时走它。

显存 / 分块档位不用管：节点 init 时已自探并写进模型默认值。

## 覆盖能力

- **调用**：选模型、传参、异步任务与产物、任务看板汇总
- **接入**：把新工作流注册进网关 / 摘除某个档（`models.yaml` 映射、`/admin/reload` 热加载）
- **权重**：下载、目录摆放、`Value not in list` 排障
- **文档核对**：文档说法与实现一致性审计
- **版本管理**：`tools/release.py bump` 统一 bump（pyproject + API.md + skills manifest + 全套测试）

## 目录结构

```
roundabout-skill/
├── SKILL.md                    # 入口：调用方式、三条易踩机制、生效分档、开工前清单
└── references/
    ├── usage.md                # 调用面：模型、参数、异步任务、产物、任务看板
    ├── add-model.md            # 接入面：注册新工作流、摘档、生效方式、测试清点
    ├── weights.md              # 权重下载 / 目录 / 排障
    └── doc-audit.md            # 文档与实现一致性核对
```

## 何时使用

- 用户说「roundabout」「走网关生成」「generate_image」「generate_video」「MCP 生图/生视频」「`/v1/images/generations`」「注册工作流」「models.yaml 加一个模型」「权重下不下来」等
- 需要驱动装了本节点的 ComfyUI 出图出视频

**不适用**：H3 提示词写法与能力边界（走 `h3-playbook-skill`）；Qwen-Image-2.1 提示词（走 `qwen-image2.1-prompt-writing-skill`）。

## 三条最容易踩的机制

1. **`workflow_overrides` 是通用路径注入器**，与参数白名单无关：`{"910.inputs.scale": 2.0}` 直接改节点，路径不存在会 400。
2. **字段注入三道闸（白名单 → `bindings` → `values`），缺一道静默无效**——只写 `defaults` 不配 binding ＝双重死键；改完 reload 后翻日志查 `unknown binding key`。
3. **找不到落点先看提交图**：`get_workflow` 或产物内嵌提交图；判「实际跑了什么」只看 `/history` 提交图、日志 `N/M` 行、产物 metadata，别信文件名 / tag。

**顺序铁律**：新工作流先在 ComfyUI 原生 `/prompt` 验通，再落 `models.yaml`。

## 生效分档

| 改了什么 | 生效方式 |
|---|---|
| `models.yaml` / `workflows/*.json` | `POST /admin/reload` 热加载 |
| `gateway/*.py` · `mcp_server.py` · `__init__.py` | 必须重启 ComfyUI |

## 版本

`skill_version: 1.18.5`
