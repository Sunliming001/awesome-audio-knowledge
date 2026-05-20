# 数字音频基础 (Digital Audio Fundamentals)

数字音频是将连续的模拟声波信号通过采样、量化和编码，转化为计算机可处理的离散二进制数据的过程。本章涵盖音频数字化的数学原理、工程实践以及在 Android/嵌入式系统中的关键概念。

---

## 1. ADC/DAC 转换链

```
模拟世界                    数字世界                    模拟世界
  │                           │                           │
  ▼                           ▼                           ▼
声波 → 麦克风 → 前置放大 → Anti-Aliasing → ADC → DSP → DAC → 重建滤波 → 功放 → 扬声器
                             滤波器              处理        (Reconstruction)
                           (LPF < fs/2)                      LPF

ADC (Analog-to-Digital Converter):
  1. 抗混叠滤波 (Anti-Aliasing Filter)
  2. 采样 (Sample)
  3. 量化 (Quantize)
  4. 编码 (Encode → PCM)

DAC (Digital-to-Analog Converter):
  1. 解码 PCM
  2. 零阶保持 (Zero-Order Hold)
  3. 重建低通滤波
  4. 模拟输出

现代 ADC/DAC 类型:
  Sigma-Delta (ΣΔ): 高过采样 + 噪声整形, 音频领域主流
  SAR (Successive Approximation Register, 顺序逼近寄存器): 快速, 中等精度, 用于控制/传感
  Pipeline: 高速, 视频/通信应用
```

---

## 2. 采样 (Sampling) 与奈奎斯特定理

### 2.1 奈奎斯特-香农采样定理

$$f_s > 2 f_{max}$$

为无失真重建最高频率 $f_{max}$ 的信号，采样率 $f_s$ 必须大于 $2 f_{max}$。

### 2.2 混叠 (Aliasing)

```
混叠示意:

  原始信号频谱:
    │     ╱╲
    │    ╱  ╲
    │   ╱    ╲
    │──╱──────╲──────────────→ f
    0        fmax   fs/2   fs

  采样后 (fs 不足, 即 fs/2 < fmax):
    │     ╱╲  ╲╱╲  ← 混叠分量折返!
    │    ╱  ╲╱╱  ╲
    │   ╱   ╱╲    ╲
    │──╱──╱────╲───╲────────→ f
    0   fs/2    fs

  防护:
    采样前必须加 Anti-Aliasing LPF (截止频率 < fs/2)
    数字系统中需要 Digital LPF (用于降采样)
```

### 2.3 常用采样率

| 采样率 | 应用 | Nyquist 频率 | 说明 |
|:---|:---|:---|:---|
| **8 kHz** | 电话 (G.711) | 4 kHz | 仅语音 |
| **16 kHz** | VoIP / 语音识别 | 8 kHz | 宽带语音 |
| **44.1 kHz** | CD 音乐 | 22.05 kHz | 历史标准 (来自 PCM-1600 视频编码) |
| **48 kHz** | 专业音频 / 视频 / Android 默认 | 24 kHz | 与视频帧率整除兼容 |
| **96 kHz** | Hi-Res Audio | 48 kHz | 高解析度 |
| **192 kHz** | 录音棚 / 母带制作 | 96 kHz | 超高解析度 |
| **384 kHz** | DSD-DoP 转换 | 192 kHz | 极少使用 |

### 2.4 采样率转换 (SRC, Sample Rate Conversion)

```
SRC 在音频系统中的场景:
  - App 输出 44.1kHz, HAL 要求 48kHz → 上采样
  - 蓝牙 Codec (16kHz) 到系统 (48kHz) → 上采样
  - 录音 48kHz 保存为 16kHz → 下采样

SRC 方法:
  1. 整数比 (L/M): 先上采样 L 倍 (插零+LPF), 再下采样 M 倍 (LPF+抽取)
  2. 多相滤波 (Polyphase Filter): 高效实现
  3. 非整数比: ASRC (Asynchronous SRC), 用于时钟不同步场景

质量等级:
  线性插值:    快速, 低质量, 引入混叠
  多相 FIR:   标准质量, Android AudioFlinger 使用
  sinc 插值:  最高质量, 计算量大
  
Android SRC 位置:
  AudioFlinger → AudioMixer → mResampler (多相 FIR)
  AudioFlinger 默认输出采样率: 48kHz
```

---

## 3. 量化 (Quantization) 与位深

### 3.1 SQNR (Signal-to-Quantization-Noise Ratio, 信号与量化噪声比) 与动态范围

$$\text{SQNR} \approx 6.02N + 1.76 \text{ dB}$$

| 位深 | SQNR | 动态范围 | 应用 |
|:---|:---|:---|:---|
| **8-bit** | 50 dB | 48 dB | 电话 (µ-law/A-law) |
| **16-bit** | 98 dB | 96 dB | CD, 一般音频 |
| **24-bit** | 146 dB | 144 dB | 专业录音/混音 |
| **32-bit float** | 1528 dB (理论) | ~150 dB (实际) | DSP 内部处理 |

### 3.2 定点 vs 浮点

```
定点 (Fixed-Point):
  格式: Q1.15 (16-bit), Q1.31 (32-bit)
  范围: [-1.0, +1.0) 或 [0, 2^N - 1]
  优势: 硬件简单, DSP/ASIC (Application-Specific Integrated Circuit, 专用集成电路) 常用
  劣势: 动态范围有限, 需要手动管理溢出
  
  Q1.15 示例:
    0x7FFF = +0.999969  (最大正值)
    0x8000 = -1.000000  (最小负值)
    0x0000 = 0.000000

浮点 (Floating-Point):
  格式: IEEE 754 float32
  结构: 1 bit 符号 + 8 bit 指数 + 23 bit 尾数
  优势: 巨大动态范围, 不易溢出
  劣势: 硬件复杂, 功耗高
  
  Android 趋势:
    AudioFlinger 混音: float32 (Android 5.0+)
    AAudio/Oboe: 推荐 AAUDIO_FORMAT_PCM_FLOAT
    HAL 接口: 支持 float32
```

### 3.3 抖动与噪声整形

```
抖动 (Dithering):
  问题: 小信号量化 → 产生与信号相关的谐波失真
  方案: 量化前加入随机噪声 (幅度 ~1 LSB)
  
  类型:
    RPDF (Rectangular Probability Density Function, 矩形概率密度): ±0.5 LSB, 消除失真相关性
    TPDF (Triangular Probability Density Function, 三角概率密度): ±1 LSB, 完全消除失真
    Shaped Dither: 结合噪声整形
  
噪声整形 (Noise Shaping):
  原理: 利用反馈将量化噪声移到人耳不敏感的高频区域
  应用: 
    - Sigma-Delta ADC/DAC 的核心原理
    - 24-bit → 16-bit 高质量位深转换
    - DSD (1-bit, 高度过采样 + 噪声整形)
```

---

## 4. PCM 数据结构

### 4.1 数据格式

```
常见 PCM 格式标识 (ALSA/Android):

  S16_LE:  Signed 16-bit, Little-Endian     (CD 标准)
  S24_LE:  Signed 24-bit, Little-Endian     (专业音频)
  S24_3LE: Signed 24-bit packed (3 bytes)   (节省空间)
  S32_LE:  Signed 32-bit, Little-Endian     (HAL 内部)
  FLOAT_LE: IEEE 754 Float, Little-Endian   (AudioFlinger)

内存布局 (S16_LE, 立体声):

  Interleaved (交织):
    [L0_lo][L0_hi][R0_lo][R0_hi][L1_lo][L1_hi][R1_lo][R1_hi]...
    └── Frame 0 ──┘└── Frame 0 ──┘└── Frame 1 ──┘└── Frame 1 ──┘
    
  Non-interleaved (非交织/平面):
    Buffer L: [L0_lo][L0_hi][L1_lo][L1_hi][L2_lo][L2_hi]...
    Buffer R: [R0_lo][R0_hi][R1_lo][R1_hi][R2_lo][R2_hi]...
    
    优势: SIMD/NEON 并行处理效率更高
    Android: AudioFlinger 混音器内部使用 non-interleaved float
```

### 4.2 码率计算

$$\text{Bitrate} = f_s \times N \times C$$

| 格式 | 采样率 | 位深 | 声道 | 码率 | 每分钟大小 |
|:---|:---|:---|:---|:---|:---|
| 电话 PCM | 8kHz | 16-bit | 1 | 128 kbps | ~0.96 MB |
| CD 音质 | 44.1kHz | 16-bit | 2 | 1411 kbps | ~10.6 MB |
| Hi-Res | 96kHz | 24-bit | 2 | 4608 kbps | ~34.6 MB |
| 录音棚 | 192kHz | 32-bit | 2 | 12288 kbps | ~92.2 MB |

---

## 5. 音频时钟与同步

### 5.1 时钟架构

```
数字音频系统的时钟层级:

  Master Clock (MCLK):
    频率: 通常为采样率的 256× 或 512×
    例: 48kHz × 256 = 12.288 MHz
    来源: 晶振 (XO) 或 PLL
    
  Bit Clock (BCLK / SCLK):
    频率: 采样率 × 位深 × 声道数
    例: 48kHz × 16 × 2 = 1.536 MHz
    
  Frame Clock (LRCLK / WCLK / FS):
    频率: = 采样率
    例: 48kHz
    作用: 标识左/右声道 (I2S) 或帧起始 (TDM)
    
  时钟关系:
    MCLK = K × LRCLK  (K = 256/512/384)
    BCLK = N × LRCLK  (N = 位深 × 声道数)
    LRCLK = fs
```

### 5.2 Jitter (时钟抖动)

```
Jitter 定义:
  采样时间点相对理想位置的偏差

  影响:
    ΔT (时间偏差) → 引入噪声底
    对于正弦信号: SNR_jitter ≈ -20 × log₁₀(2π × f × Δt_rms)
    
  示例 (48kHz 系统):
    Jitter = 1 ns rms → SNR ≈ 130 dB @ 20kHz (可忽略)
    Jitter = 1 µs rms → SNR ≈ 70 dB @ 20kHz (严重!)
    
  Jitter 来源:
    - PLL 锁相环噪声
    - 电源纹波
    - 数字信号串扰
    
  减少 Jitter:
    - 使用低噪声晶振
    - 时钟 reclocking (在 DAC 前重新采样)
    - 异步 USB (ASRC + 本地时钟)
```

---

## 6. 常见音频术语速查

| 术语 | 定义 | 典型值 |
|:---|:---|:---|
| **帧 (Frame)** | 所有声道在同一采样时刻的数据 | S16 立体声: 4 bytes |
| **周期 (Period)** | HAL 一次处理的帧数 | 240 frames (5ms @ 48kHz) |
| **缓冲区 (Buffer)** | 包含多个周期的环形缓冲 | 960 frames (20ms) |
| **延迟 (Latency)** | 数据从输入到输出的时间 | 10-200ms |
| **Underrun** | 播放缓冲区空 (数据不足) | 爆音/卡顿 |
| **Overrun** | 录音缓冲区满 (数据未读) | 数据丢失 |
| **dBFS** | 相对于数字满幅的分贝值 | 0 dBFS = 最大值 |
| **Headroom** | 距离数字削波的余量 | 通常留 3-6 dB |

---

## 7. 关键参考 (References)

1.  *Digital Audio Signal Processing* - Udo Zölzer
2.  [The Science of Sample Rates - Xiph.org](https://xiph.org/video/vid2.shtml)
3.  [Dither and Noise Shaping - Wikipedia](https://en.wikipedia.org/wiki/Dither)
4.  [Understanding Jitter - Analog Devices](https://www.analog.com/en/technical-articles/understanding-jitter.html)
5.  [PCM Format Reference - ALSA Project](https://www.alsa-project.org/alsa-doc/alsa-lib/pcm.html)

---
*Next Module: [02. 硬件系统 (Hardware System)](../02-Hardware-System/README.md)*
