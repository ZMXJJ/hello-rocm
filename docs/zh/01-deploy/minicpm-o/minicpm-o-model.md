## MiniCPM-o 4.5 模型介绍

本文介绍 **MiniCPM-o 4.5** 的全模态架构、子模型组成与 GGUF 文件清单，帮助你在开始部署前做好准备。

> 前置条件：建议先完成 [ROCm 基础环境安装](/zh/00-environment/)。

---

### 一、什么是 MiniCPM-o

MiniCPM-o（Omni）是面壁智能（OpenBMB）推出的端侧全模态大模型系列。4.5 版本基于约 8B 参数的 LLM 骨干，集成了以下能力：

| 能力 | 说明 |
|------|------|
| **文本对话** | 与普通 LLM 相同，支持多轮对话和系统提示词 |
| **语音输入** | 实时音频流编码，理解用户语音并转为文本 |
| **图像理解** | 支持单张图片的视觉问答 |
| **语音输出（TTS）** | 用 token2wav 生成自然语音，支持音色克隆 |
| **全双工对话** | 同时监听麦克风 + 生成语音回复，类似真实电话通话体验 |

相比同参数量的纯文本模型，MiniCPM-o 4.5 的全双工响应延迟可达到约 **800ms**，在端侧设备上实现了接近实时的语音交互。

---

### 二、模型架构

MiniCPM-o 4.5 由多个独立模块组成，在 `llama.cpp-omni` 中以独立 GGUF 文件分别加载：

```
MiniCPM-o 4.5（全模态推理）
├── LLM（主干）          MiniCPM-o-4_5-Q4_K_M.gguf         ~4.9 GB
├── 视觉编码器           vision/MiniCPM-o-4_5-vision-F16.gguf  ~0.9 GB
├── 音频编码器           audio/MiniCPM-o-4_5-audio-F16.gguf    ~1.2 GB
├── TTS 语言模型         tts/MiniCPM-o-4_5-tts-F16.gguf        ~0.5 GB
├── TTS 投影层           tts/MiniCPM-o-4_5-projector-F16.gguf  ~0.1 GB
└── Token-to-Wav（声码器）
    ├── token2wav-gguf/encoder.gguf
    ├── token2wav-gguf/flow_matching.gguf
    ├── token2wav-gguf/flow_extra.gguf
    ├── token2wav-gguf/hifigan2.gguf
    └── token2wav-gguf/prompt_cache.gguf    合计 ~0.7 GB
```

**全部 10 个 GGUF 文件合计约 8.3 GB**（Q4_K_M 量化版本）。

---

### 三、可用版本与量化

本教程以 **Q4_K_M 量化**为主，该量化方案在精度和显存占用之间取得了良好平衡，适合在消费级 AMD GPU 和 APU 上运行。

| 文件 | 量化 | 用途 |
|------|------|------|
| `MiniCPM-o-4_5-Q4_K_M.gguf` | Q4_K_M | LLM 主干（文本生成） |
| `vision/MiniCPM-o-4_5-vision-F16.gguf` | F16 | 图像理解 |
| `audio/MiniCPM-o-4_5-audio-F16.gguf` | F16 | 语音输入理解 |
| `tts/MiniCPM-o-4_5-tts-F16.gguf` | F16 | 语音合成（LM 部分） |
| `tts/MiniCPM-o-4_5-projector-F16.gguf` | F16 | 语音合成（投影层） |
| `token2wav-gguf/*.gguf`（5 个文件） | — | 声码器（音频波形生成） |

> 子模型（视觉、音频、TTS）目前仅有 F16 版本，因此即使 LLM 主干做了 Q4 量化，总体显存占用仍约 8.3 GB。未来如有 Q8 / Q4 子模型版本，显存需求可进一步压缩。

---

### 四、显存需求估算

| 运行模式 | 需要加载的模块 | 约 VRAM |
|----------|----------------|---------|
| 纯文本对话 | LLM | ~5 GB |
| 文本 + 图像理解 | LLM + 视觉编码器 | ~6 GB |
| 文本 + 语音输入 | LLM + 音频编码器 | ~6.5 GB |
| 全模态（语音输入 + TTS 输出） | LLM + 视觉 + 音频 + TTS + Token2Wav | **~9 GB** |

> **注意**：上述估算基于 4096 token 上下文。使用 8192 token 上下文时，KV Cache 会额外占用约 1–2 GB。
>
> AMD Ryzen AI MAX+ 395 等 APU 的统一内存（64 GB）可全部作为 VRAM 使用，非常适合运行全部模块，并保留充足余量应对大上下文。

---

### 五、对话模式说明

`llama.cpp-omni` 和 MiniCPM-o-Demo 支持以下 4 种对话模式：

| 模式 | 输入 | 输出 | 特点 |
|------|------|------|------|
| **Turn-based（轮询）** | 音频文件 / 文本 | 文本 + TTS | 最稳定，适合验证基础功能 |
| **Half-Duplex（半双工）** | 实时麦克风 | 文本 + TTS | 说完后等待，无打断 |
| **Omni（全双工）** | 实时麦克风 + 摄像头 | 实时语音流 | 同时听+说，有打断能力 |
| **Audio Duplex（纯语音全双工）** | 实时麦克风 | 实时语音流 | 同 Omni 但不用摄像头 |

CLI 部署（`llama-omni-cli`）适合验证 Turn-based 模式；Web Demo 支持全部 4 种模式，且可在浏览器中通过摄像头/麦克风实时体验 Omni 全双工。

---

### 六、模型下载来源

GGUF 文件可从 **ModelScope**（国内推荐，速度更快）或 **Hugging Face** 下载：

| 来源 | 地址 |
|------|------|
| Hugging Face | [OpenBMB/MiniCPM-o-4_5-gguf](https://huggingface.co/OpenBMB/MiniCPM-o-4_5-gguf) |
| ModelScope | [OpenBMB/MiniCPM-o-4_5-gguf](https://www.modelscope.cn/models/OpenBMB/MiniCPM-o-4_5-gguf) |

---

### 参考资源

- [MiniCPM-o 官方仓库](https://github.com/OpenBMB/MiniCPM-o)
- [MiniCPM-o 技术报告（arxiv）](https://arxiv.org/abs/2408.01800)
- [llama.cpp-omni](https://github.com/tc-mb/llama.cpp-omni)

---

> 了解完模型架构后，接下来请前往 [llama.cpp-omni CLI 部署](./llamacpp-omni-rocm7-deploy.md) 或 [Web Demo 全双工部署](./webdemo-rocm7-deploy.md)。
