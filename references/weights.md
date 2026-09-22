# 权重：下哪些、怎么下、放哪里

Roundabout **不含任何权重**。内置工作流要跑起来，得先把权重备齐。

## 规模速查

| 口径 | 数量级 |
|---|---|
| 内置工作流 | 图像档 / 视频档 / 工具档三族（支数以 `workflows/` 实际文件与 `models.yaml` 为准） |
| 实际引用的权重文件 | 二十上下 |
| 合计 | 约 200 GB 量级（图像档约 77 GB / 视频档约 120 GB） |
| 只想先跑图像 | **约 77 GB**，可以先只下这一族 |

**权威清单在 `README.md` 的「权重清单」节**（逐文件的体积 / 目标目录 / HF repo / repo 内路径 +
现成的 `curl` 命令），且**引用数要用脚本遍历 `workflows/*.json` 枚举**，别沿用文档里的旧数。
本节只记「怎么下不踩坑」。

例外：`workflows/example_txt2img.json` 是接入样本，用你自己的 checkpoint，**不计入引用数**。

## 三个必知的坑

### 1. 有一个文件必须下载后改名

| 上游文件名 | 工作流引用的名字 |
|---|---|
| `minimax_h3_latent_upscaler_3d_conv_v1_fp16.safetensors` | `minimax_h3_latent_upscaler_3d_fp16.safetensors`（去掉 `_conv_v1`） |

其余文件的下载名 = 使用名。**改名是硬要求**——工作流 JSON 里写的是后者。

### 2. 国区走镜像，但 *repo 内路径* 不变

```bash
set HF_ENDPOINT=https://hf-mirror.com          # Windows
export HF_ENDPOINT=https://hf-mirror.com       # Linux / macOS
```

repo ID 与 repo 内路径完全一致，只换端点。`curl` 形式的下载命令里把 `$B` 设为端点变量即可。

> **大文件优先走 ModelScope（Comfy-Org 系仓库两边都在）。** 对 Comfy-Org 官方仓库实测：
> `hf-mirror` 在几 GB 级 `safetensors` 上**单连接吞吐会持续衰减、偶发超时**（分片并发也只把总量
> 抬到个位数 MB/s 量级），而 ModelScope 同 repo 能稳定跑满带宽。URL 形态把域名与路径段换掉即可：
>
> ```bash
> # HF 镜像：   $B/<repo>/resolve/main/<repo 内路径>
> # ModelScope: https://modelscope.cn/models/<repo>/resolve/master/<repo 内路径>
> ```
>
> ModelScope 支持 `Range` 请求，适合**分片下载 + 断点续传**（每片写独立的 `.partN`，齐了再顺序拼接，
> 单次超时不用从头再来）。
>
> ⚠️ **别用「文件大小对了」判断下载完成**：不少下载器（含 `hf_transfer` 一类）会**预分配**到
> 目标字节数，半成品也是完整大小。要么下完自己记一个校验标记，要么按分片字节数核对。

### 3. `hf download` 会保留仓库目录结构 —— 所以默认给 `curl`

Comfy-Org 的仓库用 `split_files/` 前缀，**那不是 ComfyUI 的目录**：

```bash
hf download Comfy-Org/MiniMax-H3 diffusion_models/minimax_h3_fl2va_int8_convrot.safetensors --local-dir models
# 下完还得手动把 split_files/... 挪到 models/diffusion_models/ 之类的位置
```

所以 README 一律给 `curl`（落到指定路径，一步到位）。用 CLI 也行，但要记得手动挪。

## 目录映射（放错就报 `value not in list`）

| 目标目录 | 放什么 |
|---|---|
| `models/diffusion_models/` | 主模型（UNET / DiT） |
| `models/text_encoders/` | 文本编码器 |
| `models/vae/` | VAE（视频档另有**音频** VAE） |
| `models/loras/` | LoRA 与蒸馏加速权重 |
| `models/latent_upscale_models/` | lift 两支的 latent 上采样权重 |
| `models/background_removal/` | BiRefNet 抠图 |

**缺文件的报错形态**：`value not in list: <字段>: <文件名>` —— 直接照着这个文件名去补即可。

## 量化变体：选错在旧卡上直接跑不了

- 视频/VLM 侧大量用 `int8_convrot` —— 它在 Ada / Ampere 上都能跑，是**通用解**。
- `nvfp4_awq` 系列**需要 Blackwell**（50 系）。在 Ada（SM 8.9）卡上要换成 `int8_convrot`。
- 想省显存可以换更小的量化版（`nvfp4` / `pruned_int8_convrot` 等，同仓库同目录下就有），
  但**必须同步改工作流 JSON 里的文件名** —— 权重名与 JSON 是硬绑定。

## pruned 与完整版不能互换

- `fasth3` / `fasth3-edit` 用 `fastvideo_fasth3_8step_v2_**pruned**_int8_convrot.safetensors`；
- `minimax-h3` 系列（含 lift 两支）用**完整版** `minimax_h3_*_int8_convrot.safetensors`。

两者不能互换，**LoRA 也会跟着不匹配**：`*_pruned_*.safetensors` 是给 curve-form 权重转换的，
挂到完整版权重上会 key 不匹配 → **LoRA 静默不加载**（不报错，只是没效果）。

## LoRA：注意命名体系

- **`alibaba-pai/MiniMax-H3-Acc-LoRAs` 里的 Acc LoRA 是 diffusers 命名**
  （`transformer_blocks.*`），ComfyUI 普通加载器只认 `diffusion_model.blocks.*` ⇒
  **key 全 warning、零 patch**，等于没挂。要走 PDD 专用节点（`MiniMaxH3PDDAccApply`）。
- 能直接用普通加载器的是 **ComfyUI 命名**的那批，在 `Comfy-Org/MiniMax-H3` 的 `loras/` 下，
  名字带 `_comfyui_` 标识（如 `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors`）。
- **社区转换版要当心**：有转换版把端点适配器 `endpoint_time_embedder.*` 并进了
  `time_embedder.proj_in/proj_out`，**端点条件进不了模型**（画面出现非单段式晕开）。
  要完整效果得上游原版 + 专用节点包。

## 共用与可替换关系（省下载量）

- `qwen3vl_8b_fp8_scaled.safetensors` 被 Boogu 全系与 `flux2-klein-image-edit-turbo` **共用**，
  下一个文件够多个模型用。
- ⚠️ **同名不同文件**：Qwen-Image 2.1 的文本编码器叫 `qwen3vl_8b_int8_convrot.safetensors`，与上面那个 `qwen3vl_8b_fp8_scaled.safetensors` **只是精度后缀不同、来自不同仓库**，别互相顶替（工作流按文件名引用，顶替了不一定报错）。同仓库还有一路 `qwen3.5_9b_qwen_image_2.1_pe_{t2i,i2i}.int8_convrot`「PE」编码器，内置工作流不用。
- `ae.safetensors`（fp32 0.34 GB）与 `flux1_vae_bf16.safetensors`（bf16 0.17 GB）是**同一个
  FLUX.1 Autoencoder 的两种精度**，按文件名被不同工作流引用 —— 名字不同就必须都存在。
  只跑 `boogu-image-edit` 而没跑 `z-image` 时，可以把 bf16 那份复制一份改名成 `ae.safetensors` 用
  （同架构可换，代价是精度）。
- `flux-2-klein-9b-kv-fp8.safetensors` 来自 **Black Forest Labs 官方仓库**（不在 Comfy-Org）；
  Comfy-Org 只提供了它的 VAE 与文本编码器仓库（`vae-text-encorder-for-flux-klein-9b`，官方拼写如此）。

## 下完自查

1. 文件名与工作流 JSON 里引用的一致（尤其改名那一个、以及量化后缀）。
2. 放进的是**对应子目录**（`value not in list` 就是这里错了）。
3. 视频档别忘**音频 VAE**（`vae/minimax_h3_audio_vae_fp32.safetensors`）——漏了会在解码段失败，
   与「权重没下」的表现不一样，容易误判成别的问题。
