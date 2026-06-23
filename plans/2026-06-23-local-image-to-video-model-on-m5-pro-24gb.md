# 本地"文字 + 图片 → 视频"模型选型与落地计划

> 目标设备：Apple M5 Pro（15 核 CPU + GPU with Neural Accelerators） · 24 GB 统一内存 · macOS 26.5 · 约 747 GB 可用磁盘
> 目标用途：可商用的本地（on-device）图生视频 / 图+文生视频（I2V / TI2V）
> 编写日期：2026-06-23

---

## 1. 结论先行（TL;DR）

| 优先级 | 推荐模型 | 许可证 | 商用是否需付费 | 24 GB 统一内存可行性 | 主要承运栈 |
| --- | --- | --- | --- | --- | --- |
| **首选** | **Wan 2.2 TI2V-5B**（Alibaba Wan-AI） | Apache-2.0 | **完全免费**（含商用） | 可行（BF16 ~10 GB 主干 + 文本编码器可 offload 到内存/SSD） | ComfyUI（MPS）/ Diffusers（MPS）|
| 备选 1（追求速度） | **LTX-Video 2.3 distilled**（Lightricks） | LTX-2 Community License | < 1000 万美元年营收免费；超过需另购商业许可 | 较好（已有原生 Apple Silicon / MLX 路径，Mac 桌面应用） | LTX Desktop / ComfyUI-LTXVideo |
| 备选 2（生态最稳） | **CogVideoX-5B-I2V**（THUDM/智谱） | Apache-2.0 | 完全免费（含商用） | 可行（5B 量级，Diffusers 官方支持 MPS） | 🤗 Diffusers / ComfyUI |
| 不推荐 | Wan 2.2 I2V-A14B（14B MoE） | Apache-2.0 | 免费 | **吃力**：BF16 ≈ 30 GB；FP8 在 Mac 上无原生支持；GGUF Q3/Q4 质量损失明显 | — |
| 不推荐 | HunyuanVideo-I2V（13B） | Tencent Hunyuan Community License | 名义免费但限制多 | 吃力 + MPS 路径尚不稳定 | — |
| 不推荐 | Stable Video Diffusion (SVD) | Stability AI Community License | 商用需付费会员 | 可行 | — |

**最终结论：在 M5 Pro / 24 GB 这台机器上做合规商用的本地"文字 + 图片 → 视频"，第一推荐 _Wan 2.2 TI2V-5B_（BF16）。** 它在"许可证宽松度（Apache-2.0，对你输出的内容不主张任何权利）× 显存友好度（5B + 高压缩 VAE，最低 8 GB 显存即可跑 480P）× 质量（官方支持 720P@24fps，单模型同时具备 T2V 与 I2V 能力）"三个维度上对本机最契合。

---

## 2. 硬件画像与边界条件

- **统一内存 24 GB**：CPU/GPU 共享，无独立 VRAM。MPS（Metal Performance Shaders）后端默认有 watermark 限制，需要通过 `PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0` 解锁，让 MPS 在需要时可使用接近全部统一内存。
- **数值精度**：
  - **FP8** 在 Apple Silicon 上**不可用**（依赖 NVIDIA Tensor Cores），即便 ComfyUI 声称支持也无法在 MPS 上跑通。本地必须使用 **BF16 / FP16** 权重，或选 GGUF 量化但要走 CPU 路径（速度差）。
  - **BF16** 是 Apple Silicon 上原生支持的 dtype，是首选格式。
- **文本编码器**（UMT5-XXL ~11 GB / Gemma 3 12B 等）必须可以 offload 到系统内存或分块加载，否则 24 GB 无法装下"文本编码器 + 扩散主干 + VAE + 激活值"。
- **磁盘 747 GB**：充裕，可容纳多个 5–14B 模型与 LoRA。
- **macOS 26.5**：需要 PyTorch ≥ 2.6（推荐 nightly 或 2.7+）以获得最新 MPS 算子；ComfyUI 推荐 ≥ 0.3.x；Python 3.11–3.13。

---

## 3. 候选模型深度对比

### 3.1 Wan 2.2 TI2V-5B —— 首选

- **来源**：Alibaba Wan-AI（[Wan-Video/Wan2.2](https://github.com/Wan-Video/Wan2.2)）。
- **能力**：单一 checkpoint 同时支持 T2V 与 I2V；720P@24fps；高压缩 VAE（16×16×4），是当前最快的 720P@24fps 开源模型之一。
- **许可证**：**Apache-2.0**。官方明确："对你生成的内容不主张任何权利"，你可以自由商用，前提是遵守许可证关于合法、不伤害他人的常规条款。
- **资源占用**（参考 ComfyUI 生态数据）：

  | 精度 | 480P 最低显存 | 720P 推荐显存 |
  | --- | --- | --- |
  | FP16/BF16 | ~8 GB | ~12 GB |
  | FP8（Mac 不可用） | ~6 GB | ~8 GB |

  扩散主干 + UMT5-XXL（offload 到系统内存）+ VAE，在 24 GB 统一内存上跑 480P~720P 短片（约 49–81 帧）是可行区间。

- **Mac 落地路径**：
  1. ComfyUI ≥ 0.3.x（内置 Wan 节点）+ `ComfyUI-WanVideoWrapper` 自定义节点。
  2. Diffusers `WanImageToVideoPipeline`，`device="mps"`，`torch_dtype=torch.bfloat16`。
- **风险点**：
  - 单次推理时长长（M5 Pro 实测预估：480P × 49 帧 ≈ 5–15 分钟/段，720P 更长）。需要做预设、批处理与冷启动缓存。
  - VAE 解码是显存峰值，需要启用 ComfyUI 的 `--fp16-intermediates` 与 tiled VAE。

### 3.2 LTX-Video 2.3 —— 备选（追求速度 / 已有 Mac 原生路径）

- **来源**：Lightricks（[Lightricks/LTX-Video](https://github.com/lightricks/ltx-Video)）。
- **能力**：扩散 Transformer，主打**实时/近实时**生成；T2V + I2V + 关键帧 + 视频续写；2B / 13B 多档；2.3 版本提供 distilled 与 NVFP4/FP8/GGUF 等多量化档。
- **许可证**：**LTX-2 Community License Agreement（非 Apache-2.0）**。
  - 年营收 **< 1000 万美元** 的个人/实体：**免费商用**。
  - 年营收 ≥ 1000 万美元的"Commercial Entity"：必须向 Lightricks 单独购买 Commercial Use Agreement，否则违约赔偿为应付许可费的两倍。
  - 该许可证还附带不与 Lightricks 竞争、属地合规等条款；分发衍生作品时需带原始许可证。
- **Mac 落地优势**：
  - 官方提供 **`ltx-video-mac` 原生应用**，并有 MLX 移植路径（Apple Silicon 原生加速框架），社区报告 M2/M3 Max 上可用，M5 Pro 性能更佳。
  - 在 ComfyUI 中通过 `ComfyUI-LTXVideo` 节点也可运行；MPS 路径稳定性已被官方列入 release notes（PyTorch ≥ 2.3 起）。
- **风险点**：
  - **许可证不是纯开源**：如果未来公司营收触发 \$10M 门槛，必须切换到付费许可，需事先在合规层面知会。
  - C2PA Content Credentials 默认未嵌入，欧盟 AI Act Article 50（2026-08-02 起执行）的合规需要自行追加水印/出处。
  - LTX Desktop on macOS 当前部分功能仍然走云端 API，要确认所选工作流是"本地推理"还是"云端调用"。

### 3.3 CogVideoX-5B-I2V —— 备选（生态最稳）

- **来源**：清华 / 智谱（THUDM/CogVideoX）。
- **能力**：5B 参数 I2V，分辨率 720×480，约 6 秒 49 帧，时序一致性较稳。
- **许可证**：**Apache-2.0**，与 Wan 2.2 同档免费商用。
- **Mac 落地**：🤗 Diffusers 已官方支持 `CogVideoXImageToVideoPipeline`，在 MPS 上跑通成本最低；社区 ComfyUI 节点齐全。
- **取舍**：质量与时长不及 Wan 2.2 TI2V-5B，但代码生态、文档、教程最齐全，作为 **fallback** 与 **早期联调对照基线** 价值高。

### 3.4 被淘汰的选项

- **Wan 2.2 I2V-A14B（14B MoE）**：质量最高，但 BF16 ~30 GB 超过统一内存；FP8 在 Mac 上不可用；GGUF Q3/Q4 在 MPS 后端的可靠性、速度都不理想。除非升级到 64 GB 内存的 Mac Studio，否则在 M5 Pro 24 GB 上不建议作为主力。
- **HunyuanVideo-I2V（Tencent）**：质量优秀，但 **许可证有强地域与人群限制**：
  - 不得在 **欧盟 / 英国 / 韩国** 使用、分发、显示模型或其产出。
  - 月活 > 1 亿需单独申请许可。
  - 不得用其产出训练任何"非 Hunyuan 系"模型。
  - 同时社区报告其 MPS 路径尚不稳定。
  对一家希望在多个司法管辖区销售产品的商业项目而言，这条许可证是显著约束，**不建议**作为主力。
- **Stable Video Diffusion (SVD)**：商用需 Stability Membership 付费授权，对本项目目标"本地、可商用、零授权费"不匹配。
- **闭源云模型（Sora 2 / Veo 3 / Kling 2.x / Runway Gen-4 / Pika 2.x）**：不满足"本地"约束，排除。

---

## 4. 推荐落地方案

### 4.1 软件栈

```
macOS 26.5
└── Homebrew
    ├── python@3.12
    ├── git, git-lfs, ffmpeg
    └── miniforge / uv（管理虚拟环境）
└── 项目目录 ~/ai/video-gen
    ├── ComfyUI（首选 UI / 主管道）
    │   ├── PyTorch 2.7+（MPS 后端，BF16）
    │   ├── ComfyUI-Manager
    │   ├── ComfyUI-WanVideoWrapper（Wan 2.2 节点 + 双阶段采样器）
    │   ├── ComfyUI-VideoHelperSuite（视频导入/导出）
    │   └── ComfyUI-LTXVideo（备选 LTX 工作流）
    └── diffusers 脚本（用于无 UI 的批处理 / API 服务）
```

### 4.2 模型下载清单（首选方案）

| 文件 | 来源 | 用途 | 体积 |
| --- | --- | --- | --- |
| `wan2.2_ti2v_5B_bf16.safetensors`（或官方 fp16） | `Wan-AI/Wan2.2-TI2V-5B` on Hugging Face | 扩散主干（T2V+I2V 合一） | ~10–12 GB |
| `umt5_xxl_bf16.safetensors` | `Comfy-Org/Wan_2.1_ComfyUI_repackaged` | 文本编码器，需 offload | ~11 GB |
| `wan_2.2_vae.safetensors` | 同上 | 视频 VAE | ~250 MB |

可选辅助：

- **Wan 2.2 LoRA**（风格化、相机控制）
- **CogVideoX-5B-I2V** 作为 fallback 基线
- **LTX-Video 2.3 distilled (bf16)** 用于"快速预览—Wan 终稿"的双档工作流

### 4.3 关键运行参数（ComfyUI / MPS）

```bash
export PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0   # 允许 MPS 使用接近全部统一内存
export PYTORCH_ENABLE_MPS_FALLBACK=1          # 个别算子未实现时回落到 CPU
python main.py \
  --force-fp16 \
  --use-pytorch-cross-attention \
  --disable-smart-memory \
  --cache-ram 2
```

ComfyUI 加载器侧：

- `WanVideoModelLoader.load_device = "offload_device"`，让主干在非推理段位于 CPU/系统内存；
- 启用 tiled VAE 解码；
- 文本编码器一次性编完即释放。

### 4.4 推荐推理预设

| 用途 | 分辨率 × 帧数 | 步数 | CFG | 预估单段耗时（M5 Pro 24GB，BF16） |
| --- | --- | --- | --- | --- |
| 快速预览 | 512×512 × 25 帧（约 1s） | 20 | 5.0 | 2–4 min |
| 标准短片 | 832×480 × 49 帧（约 2s） | 30 | 6.0 | 6–12 min |
| 高清终稿 | 1280×720 × 49–81 帧 | 30–40 | 6.0 | 15–30 min |

> 上述时间为 Wan 2.1 1.3B/5B 在 M2/M3 Max 上的实测外推 + LTX-Video 在 M3 Max 上 ≈ 1/8 RTX 4090 速度的换算，仅供产品决策参考；上线前需在本机做基准测试。

---

## 5. 实施计划（按里程碑，不按日历周期）

### 里程碑 M1 — 环境搭建与基线打通

- [ ] 安装 Homebrew / git-lfs / ffmpeg / Python 3.12 / uv。
- [ ] 拉取 ComfyUI，配置 PyTorch nightly（MPS）+ ComfyUI-Manager。
- [ ] 安装 `ComfyUI-WanVideoWrapper`、`ComfyUI-VideoHelperSuite`、`ComfyUI-LTXVideo`。
- [ ] 下载 Wan 2.2 TI2V-5B（BF16）+ UMT5-XXL（BF16）+ VAE。
- [ ] 跑通官方示例 I2V 工作流：单张图 + 提示词 → 512×512 × 25 帧 webp。
- **退出条件**：能够稳定生成至少 10 段无 OOM 的视频，平均耗时记录在案。

### 里程碑 M2 — 质量与显存调优

- [ ] 引入 `PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0` + `offload_device` 组合，验证 720P × 49 帧。
- [ ] 评估 tiled VAE / `--fp16-intermediates` 对峰值显存的影响。
- [ ] 建立 3 套预设（预览 / 标准 / 终稿），将其保存为 ComfyUI workflow JSON 入仓。
- [ ] 用 CogVideoX-5B-I2V 跑相同 prompt 作为质量对照。
- **退出条件**：720P 终稿可在本机稳定产出；显存峰值 < 22 GB。

### 里程碑 M3 — 商用合规与产品化

- [ ] 在仓库内归档 `LICENSES/` 目录，留存 Wan 2.2 / UMT5 / VAE 的 Apache-2.0 文本。
- [ ] 编写《产出物使用条款》模板，明示用户对生成内容承担合规责任（许可证要求的常规约束）。
- [ ] 设计内容安全护栏：
  - 输入侧：图像/提示词的 NSFW / 政治敏感 / 名人识别检测；
  - 输出侧：可选 **C2PA Content Credentials** 写入（应对欧盟 AI Act Article 50，2026-08-02 生效）。
- [ ] 对 LTX-Video 作为加速备选时，单独评估年营收阈值与替代许可路径。
- **退出条件**：可对外提供"本地、可商用、合规"的视频生成 demo，许可证与水印链路完备。

### 里程碑 M4 — 工程化封装

- [ ] 在 `diffusers` 层封装命令行 / FastAPI 服务，暴露 `POST /i2v`：参数 = 图片 + 提示词 + 预设。
- [ ] 加入任务队列（如 `arq` + Redis）做串行/批量调度，防止 24 GB 内存被并发任务挤爆。
- [ ] 增加进度回调（步数级别）与产物 MP4/WebP 自动转码（ffmpeg）。
- [ ] 出一份《模型替换指南》，方便后续升级到 Wan 2.3 / LTX-3 / 新 5B 模型时只改配置。
- **退出条件**：开发同事可以通过 HTTP 调用本地服务获得视频，端到端无人值守。

---

## 6. 主要风险与对策

| 风险 | 说明 | 对策 |
| --- | --- | --- |
| 推理太慢 | 5B 模型在 M5 Pro 上单段 ≥ 5 分钟，不适合实时交互 | 双档工作流（LTX-Video 做预览 + Wan 2.2 做终稿）；任务队列异步化；预热文本编码器。 |
| 内存峰值溢出 | 24 GB 在 720P / 长帧数下接近上限 | 强制 BF16 + offload + tiled VAE；必要时降到 480P 或减少帧数；监控 `vm_stat`。 |
| 模型许可证误判 | 误用了 Hunyuan / SVD 等限制性许可证 | 在 `LICENSES/` 留存所有原始许可证文本；CI 检查模型清单。 |
| 欧盟 AI Act 合规 | 生成式视频需透明性标识 | 默认嵌入 C2PA Content Credentials；UI 上标注"AI 生成"。 |
| 苹果生态依赖 | 后续 macOS / PyTorch MPS 升级可能破坏算子 | 锁定 `torch` / `ComfyUI` 版本到已验证组合；引入 smoke test。 |

---

## 7. 决策记录（ADR 摘要）

- **ADR-001**：主模型选 Wan 2.2 TI2V-5B（Apache-2.0、Mac 友好、T2V+I2V 合一）。
- **ADR-002**：备选 LTX-Video 2.3 仅用于"速度优先"场景，并在合规层标注 \$10M 年营收阈值。
- **ADR-003**：排除 HunyuanVideo（territory 限制）、SVD（付费会员）以及所有 14B 量级 MoE（24 GB 不足）。
- **ADR-004**：所有推理强制 BF16；禁用 FP8 路径；ComfyUI 启动参数纳入版本控制。

---

## 8. 参考资料

- Wan-Video/Wan2.2 GitHub & HF 模型卡（Apache-2.0）
- ComfyUI 文档：Video Generation Models / Wan 节点 / LTX 节点
- Lightricks LTX-Video & ComfyUI-LTXVideo 仓库 + LTX-2 Community License Agreement
- THUDM CogVideoX & 🤗 Diffusers `CogVideoXImageToVideoPipeline`
- Tencent Hunyuan Community License Agreement（territory & MAU 条款）
- Apple PyTorch MPS 后端说明 / `PYTORCH_MPS_HIGH_WATERMARK_RATIO` 文档
- 欧盟《人工智能法案》第 50 条（生成式内容透明性，2026-08-02 起执行）
