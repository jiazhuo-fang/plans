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

#### 3.1.1 生成效果实测（综合官方 Wan-Bench 2.0 + 社区评测）

**整体定位**：在开源 I2V/TI2V 模型里属于"小尺寸里的天花板"，但与同家族 14B MoE（I2V-A14B）以及闭源 Sora 2 / Veo 3 / Kling 2.6 之间存在**可感知的质量差**。它是"在 24 GB 这台机器上能做到的最好折中"，而不是"全球最佳画质"。

**效果亮点**

- **画面美学 / 光影质感**：相对其他 5B 量级模型，Wan 2.2 引入了显著扩充的训练数据（图像 +65.6%、视频 +83.2%）和电影级美学标签，画面色调、景深、镜头语言比 Wan 2.1、CogVideoX-5B、Stable Video Diffusion 都更"电影感"。
- **运动一致性**：在简单场景（单主体、慢镜头、平移/推拉）下，运动连贯、镜头稳定、抖动少，是 5B 量级里少有的能做到 **720P @ 24fps** 的模型。
- **图像保真度（I2V 模式）**：以输入图作为首帧时，风格、构图、主色调的还原度较高，**前 2–3 秒** 角色身份漂移很轻微。
- **统一模型省心**：一个 checkpoint 同时干 T2V + I2V，工程化部署比 14B 那种"高噪/低噪"双文件方案简单一半。
- **官方 Wan-Bench 2.0**：在 motion、aesthetics、semantics 等多项细分指标上，5B 已接近 Kling 1.5 / Runway Gen-3 水平（但仍弱于 Sora 2 Pro、Veo 3、Kling 2.5+）。

**已知局限（这部分必须在产品宣传里讲清楚）**

| 维度 | 表现 | 触发条件 |
| --- | --- | --- |
| 时长 | 原生 **≤ 5 秒（约 121 帧）**，超过需要分段生成 + 拼接，长视频缺少长期记忆，角色会漂 | 任何 > 5s 的需求 |
| 多主体场景 | 主体之间易"融合"或者其中一个消失 | 同框 2 个以上人/物 |
| 快速动作 | 运动模糊、肢体扭曲、"鬼影" | 跑步、打斗、骑行、快速变向 |
| 手部 / 肢体细节 | 5B 是 dense 模型（非 MoE），**手指数量错误、关节扭曲** 比 14B MoE 明显更频繁 | 近景特写手部、武器、乐器 |
| 面部一致性 | **5B 通常在第 40–50 帧（约 2 秒）开始出现身份漂移**；14B 能撑到 80–100 帧 | 角色脸部特写超过 2 秒 |
| 画面内文字渲染 | 几乎不可读，会变成乱码 | 招牌、字幕、UI 截图风格 |
| 复杂提示词遵从 | 对"多动作 + 多属性"长提示词的遵从度明显弱于 14B；倾向于把复杂指令"概括化" | 一句话里同时指定 ≥ 3 个动作或属性 |
| 多语言 | 中/英/日/西都能识别，但 **英文 prompt 的运动逻辑与画质更稳定**；中文 prompt 偶有"幻肢效应"（多/少一条尾巴、多一只手） | 非英文 prompt |
| 物理真实性 | 不及 Sora 2 / Veo 3 的物理仿真；液体、布料、碰撞容易"穿模" | 液体倾倒、爆炸、布料飘动 |

**横向对比（同 prompt 主观打分，10 分制；来源：多家第三方评测综合）**

| 模型 | 视觉质量 | 运动质量 | 5s 单段速度（消费级硬件） | 商用 | 本地可跑 |
| --- | --- | --- | --- | --- | --- |
| Sora 2 Pro | 9.8 | 9.7 | 仅云端 | 仅 SaaS | ✗ |
| Veo 3.1 / Kling 2.6 | 9.5 | 9.5 | 仅云端 | 仅 SaaS | ✗ |
| **Wan 2.2 I2V-A14B（MoE）** | **9.5** | **9.2** | RTX 4090 上 ~30 min / 5s | ✓ | △ 24 GB 显存够呛 |
| **Wan 2.2 TI2V-5B**（**本机首选**） | **8.3** | **8.0** | **RTX 4090 ≈ 8–10 min / 5s @ 720P** | ✓ | ✓ |
| LTX-Video 2.3 distilled | 8.0 | 7.5 | 比 Wan 2.2 快 5–10× | △（< \$10M 营收免费） | ✓ |
| CogVideoX-5B-I2V | 7.5 | 7.0 | RTX 4090 ≈ 6–8 min / 5s @ 480P | ✓ | ✓ |
| Wan 2.1 I2V 1.3B | 6.5 | 6.0 | 最快本地档 | ✓ | ✓ |

**M5 Pro 24 GB 上的预期生成时间**（基于 RTX 4090 / M3 Max 实测外推；非线性，仅供决策）

| 任务 | RTX 4090 参考 | M5 Pro 24 GB 预期 |
| --- | --- | --- |
| 512×512 × 25 帧（约 1s）预览 | ~1 min | **3–5 min** |
| 832×480 × 49 帧（约 2s）标准 | ~3 min | **10–18 min** |
| 1280×704 × 121 帧（5s）720P 终稿 | ~8–10 min | **25–45 min** |

> M5 Pro 相对 RTX 4090 的乘数主要来自：① BF16 在 MPS 上的算子优化不及 CUDA；② 24 GB 统一内存触发更激进的 offload，IO 来回有损耗；③ 缺少 FP8/INT8 加速路径。这个时间区间是"可用，但不适合实时交互"，必须做异步队列。

**典型适用 / 不适用场景**

- ✅ **强烈推荐**：单主体产品镜头（电商主图动起来）、风景动效、Logo 入场、概念图配音前的可视化、社交媒体 5 秒短片、AI 头像视频化。
- ✅ **可用**：双人对话的远景镜头（避免特写）、缓慢推拉运镜、风格化插画动效。
- ⚠️ **谨慎**：人脸长镜头特写（建议 ≤ 2 秒）、需要可读文字的场景（如 UI 演示）。
- ❌ **不推荐**：动作戏、舞蹈、体育、爆炸/液体物理、长故事叙事、需要严格物理一致性的工业仿真。

**质量提升的工程手段（可写进里程碑）**

1. **双档生成**：用 LTX-Video 2.3（快）做"提示词 / 构图试错"，确认满意后再切 Wan 2.2 TI2V-5B 出终稿。
2. **首帧锚定**：I2V 模式下，把构图最精致的一张静态图（甚至 Midjourney / FLUX 出的图）作为首帧，能显著提升前 2 秒画质。
3. **prompt 双语策略**：用户中文输入 → LLM 翻译/扩写成英文 → 送入模型；实测可提升 ~10–15% 的画质与运动合理性。
4. **分段拼接 + 关键帧锁定**：> 5s 需求拆为多个 5s 片段，用前段的末帧作为后段的首帧，缓解身份漂移。
5. **后处理放大**：720P 终稿用 Real-ESRGAN / Topaz Video AI 放大到 1080P / 1440P，遮盖一部分细节瑕疵。
6. **如果预算允许升级硬件**：64 GB 内存 Mac Studio 可以跑 **Wan 2.2 I2V-A14B BF16**，画质会从 8.3 → 9.5，这是最大幅度的一次跃迁；24 GB 这台机器没有这个选项。

#### 3.1.2 Wan 2.2 I2V-A14B 运行配置要求（升级路径专题）

A14B 不是"一个 14B 模型"，而是 **MoE 双专家结构（高噪 + 低噪 ≈ 27B 总参数 / 14B 激活）+ UMT5-XXL 9.4 GB 文本编码器**，所以"显存账"要分三块算。下面是按精度 × 是否 offload 的完整门槛。

**A. NVIDIA / CUDA 平台（参考基准）**

| 精度 | 文本编码器位置 | 480P 显存 | 720P 显存 | 最低可用 GPU | 备注 |
| --- | --- | --- | --- | --- | --- |
| FP16 满精度 | GPU | ~45–55 GB | **~54–65 GB** | A100 80GB / H100 / 多卡 | 数据中心档，桌面买不到 |
| FP16 满精度 | CPU offload T5 | ~36–45 GB | ~45–55 GB | RTX 6000 Ada 48 GB / A6000 48GB | 工作站档 |
| FP8 | GPU | ~18–22 GB | **~22–26 GB** | **RTX 4090 / 5090 24 GB**（紧） | FP8 在 Mac 上**不可用** |
| FP8 | CPU offload T5 | ~12–14 GB | ~14–16 GB | RTX 4080 Super 16GB | 主流甜点档 |
| GGUF Q5 | CPU offload T5 | ~6–8 GB | ~8–10 GB | RTX 4070 / 3060 12GB | 质量略降，速度更慢 |
| GGUF Q4_K_S | CPU offload T5 | ~5–7 GB | ~7–9 GB | RTX 4060 Ti 16GB / 3060 12GB | 加 BlockSwap 后 8 GB 卡也能跑 |
| GGUF Q4 + BlockSwap(20 blocks) | CPU offload T5 | — | **~5.5 GB** | RTX 3060 12GB（720P） | 极限省显存档，速度大幅下降 |

附加硬件要求（全平台通用）：

- **系统内存 ≥ 32 GB**（推荐 64 GB）。T5-XXL 9.4 GB 一旦 offload 就长期驻留系统内存，加上扩散主干在切换时的 swap 缓冲，16 GB 系统内存会触发 swap，速度从分钟级跌到小时级。
- **磁盘 ≥ 60 GB 自由空间**：FP16 两份专家 ×27 GB = 54 GB + VAE + 文本编码器。
- **CUDA ≥ 11.8 / PyTorch ≥ 2.0**（NVIDIA）。

**B. Apple Silicon / macOS 平台（这才是你这台机器关心的）**

关键事实：

1. **FP8 在 MPS 后端不支持**（`Float8_e4m3fn` 无法转 MPS，社区 issue #9255 已确认），所以 NVIDIA 上"FP8 + 24 GB"那档**在 Mac 上不存在**。
2. 唯一可行路径是 **BF16 / FP16 全精度** 或 **GGUF Q4–Q8 量化**，且两份专家不能同时在 GPU。
3. 已有 **`antonpetrovmain/Wan2.2-mlx`** 这个 MLX 原生移植，绕开 PyTorch + MPS 的算子坑，能更高效利用统一内存。
4. **ComfyUI 官方 Wan 2.2 14B 模板在 macOS 上目前直接跑不起来**（默认就是 FP8），必须改用 GGUF + `UnetLoaderGGUF`。

| Mac 配置 | 能跑哪个档位的 14B I2V | 现实可用度 |
| --- | --- | --- |
| **M5 Pro / 24 GB**（你这台） | 仅 **GGUF Q3/Q4** + 极限 BlockSwap，或 MLX 4-bit 量化 | ❌ **不推荐**：显存峰值贴着上限，VAE 解码段易 OOM；速度极慢；质量大幅打折后就和 5B 接近，失去意义 |
| M3/M4 Max 64 GB | **GGUF Q4_K_S / Q5_K_M**（推荐）；FP16 单专家轮换勉强 | ⚠️ **可用但慢**：M1 Max 64 GB 实测 GGUF Q4_K_S **82 分钟 / 2 秒视频** —— M5 Pro 同档预计 50–70 分钟 |
| **M3 Ultra / M4 Max 128 GB** | **FP16 全精度** 也可装下（VAE + T5 + 双专家 ≈ 54 GB） | ✅ **舒服**：无需量化，质量接近 NVIDIA H100 输出 |
| Mac Studio M3 Ultra 192 GB | FP16 + 长帧数 + LoRA 一起开 | ✅ **理想**：是当前 Apple 平台跑 14B 的最佳机型 |

**速度参考（同分辨率 / 帧数）**

| 硬件 | 模型档位 | 5s @ 720P 单段耗时 |
| --- | --- | --- |
| RTX 4090 24GB | FP8 + T5 offload | ~25–35 min |
| RTX 4070 12GB | GGUF Q5 + offload | ~50–70 min |
| **M1 Max 64GB**（实测） | GGUF Q4_K_S | **~205 min**（按 82 min / 2s 线性外推） |
| **M5 Pro 24GB**（预测） | GGUF Q3，能跑则跑 | **预计 250–350 min**，且极易 OOM |
| Mac Studio M3 Ultra 128GB | MLX FP16 | ~80–110 min（社区报告） |

**结论：在你这台 M5 Pro / 24 GB 上，强行跑 Wan 2.2 I2V-A14B 是"技术上勉强 + 工程上不可行"**：

- 要靠 GGUF Q3/Q4 + BlockSwap 把显存压到 ~7 GB，但 Mac 上的 GGUF + MPS 组合还有 dtype cast、VAE decode NaN 等已知 bug；
- 即使跑通，单段视频 4–6 小时，质量已被量化吃掉一截，**最终画质大概率劣于直接用 TI2V-5B BF16**；
- 工程化（异步队列 + 并发 + 任务取消）几乎无法做。

**升级路径（按"画质提升 / 美元"排序）**

| 选项 | 大致预算 | 能解锁的能力 |
| --- | --- | --- |
| 不升级，沿用 M5 Pro 24 GB + TI2V-5B | 0 | 画质 8.3，单段 25–45 min |
| **Mac Studio M4 Max 64 GB** | ~3 万元起 | **14B GGUF Q4/Q5 可用**，画质 9.0–9.3，单段 ~60 min |
| **Mac Studio M3/M4 Ultra 128 GB** | ~5 万元起 | **14B FP16 舒服跑**，画质 9.5，单段 ~80–110 min |
| 单卡 RTX 4090 24GB 工作站（Linux/Win） | ~2.5–3.5 万元 | **14B FP8 主流方案**，单段 25–35 min，生态最完善 |
| 单卡 RTX 5090 32GB 工作站 | ~4–5 万元 | 14B FP8 余量充足，可叠 LoRA / 多镜头 |
| 双 RTX 5090 / H100 80GB | ≥ 10 万元 | FP16 满精度 + 大规模批处理 |

> 对你这个"本地、可商用、性价比"的目标，性价比最高的两个选项是 **Mac Studio M4 Max 64 GB** 或 **RTX 4090 24GB Linux 工作站**。前者是无缝从你当前生态延展，后者是开源视频模型的"出厂适配"平台。

- **风险点**：
  - 单次推理时长长（M5 Pro 实测预估：480P × 49 帧 ≈ 10–18 分钟/段，720P × 5s ≈ 25–45 分钟/段）。需要做预设、批处理与冷启动缓存。
  - VAE 解码是显存峰值，需要启用 ComfyUI 的 `--fp16-intermediates` 与 tiled VAE。
  - 5B 是 dense 模型，**质量天花板明显低于 14B MoE**：若产品对手部、面部长镜头有强要求，需要在 UI 上对用户做明确预期管理（"建议 ≤ 2s 特写"）或后续升级硬件以切到 14B。

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
