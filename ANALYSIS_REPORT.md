# Qwen3-TTS 开源项目深度源码审计与架构拆解

## Phase 1: 项目全景与文件结构

### 1.1 核心模型发布版

| 模型名称 | 参数量 | 特性 | 语言支持 | 流式 | 指令控制 |
|---------|--------|------|---------|------|---------|
| Qwen3-TTS-Tokenizer-12Hz | - | 音频编解码器 | - | - | - |
| Qwen3-TTS-12Hz-1.7B-VoiceDesign | 1.7B | 语音设计 | 10种语言 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-CustomVoice | 1.7B | 自定义音色+指令控制 | 10种语言 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-Base | 1.7B | 快速3秒克隆 | 10种语言 | ✅ | ❌ |
| Qwen3-TTS-12Hz-0.6B-CustomVoice | 0.6B | 轻量自定义音色 | 10种语言 | ✅ | ❌ |
| Qwen3-TTS-12Hz-0.6B-Base | 0.6B | 轻量克隆基座 | 10种语言 | ✅ | ❌ |

### 1.2 项目特性总结

**核心定位**: 基于离散多码本语言模型的端到端 TTS 系统

**关键创新**:
- 纯端到端架构（绕过 LM+DiT 信息瓶颈）
- 12Hz Tokenizer（每秒12个码元）
- 双轨混合流式生成（97ms 首包延迟）
- 10种语言支持
- 自然语言指令控制

## Phase 2: 架构设计与技术栈

### 2.1 代码分层
```
qwen_tts/
├── __init__.py              # 包入口
├── inference/
│   ├── qwen3_tts_model.py   # 推理封装层
│   └── qwen3_tts_tokenizer.py # 音频编解码器
└── core/
    └── models/              # 核心模型定义
```

### 2.2 核心架构
- **Backbone**: Qwen Decoder-only
- **Audio Tokenizer**: Qwen3-TTS-Tokenizer-12Hz (16个码本)
- **流式**: Dual-Track Hybrid Streaming (97ms延迟)

### 2.3 技术栈
- transformers - AutoModel/AutoProcessor 加载
- torch - PyTorch 核心框架
- flash-attn - 加速注意力计算
- librosa - 音频加载与重采样
- soundfile - 音频文件读写

## Phase 3: 工作原理

### 3.1 推理链路
**CustomVoice模式**:
```
文本 → Tokenize → 自回归生成Audio Codes → decode → 音频
```

**Voice Clone模式 (ICL)**:
```
参考音频 → encode(ref_code) → 拼接 → 生成 → decode
```

### 3.2 Prompt模式
| 模式 | 输入 | 特点 |
|-----|------|------|
| CustomVoice | speaker_id | 预定义音色 |
| VoiceDesign | instruct | 自然语言生成 |
| Base | ref_audio+text | 3秒克隆ICL |

## Phase 4: 开发者指南

### 4.1 入手
```bash
pip install qwen-tts + flash-attn
```

### 4.2 阅读顺序
1. 配置层 - qwen3_tts_config.py
2. 生成核心 - qwen3_tts_for_conditional_generation.py ⭐⭐⭐⭐⭐
3. 推理封装 - qwen3_tts_model.py

### 4.3 避坑
- FlashAttention版本兼容
- 必须使用 bfloat16/float16
- Batch大小必须匹配
- 显存: 0.6B(4-8GB), 1.7B(8-16GB)

## 总结

**Qwen3-TTS架构亮点**:
1. 纯端到端（避免 LM+DiT 级联误差）
2. 极速流式（97ms 首包延迟）
3. 灵活控制（自然语言指令 + ICL 续写）
4. 完整生态（HuggingFace + vLLM 支持）

**二次开发切入点**:
- 微调: finetuning/sft_12hz.py
- 新增音色: 扩展 speaker_id 映射
- 新语言: 修改 language embedding
