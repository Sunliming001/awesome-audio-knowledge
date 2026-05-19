# 音频软件核心概念 (Audio Software Core Concepts)

上一章从物理和数学角度讲解了数字音频原理，本章从**软件工程师的视角**梳理在 Android / Linux / 嵌入式音频开发中日常打交道的核心概念和数据结构，并配合代码示例，帮助你在阅读源码、调试问题时胸有成竹。

---

## 1. 七大核心概念一览

```
音频软件的 7 个基本量:

  ┌──────────────────────────────────────────────────────────┐
  │  Sample Rate  ──→  每秒采集/回放多少个采样点            │
  │  Bit Depth    ──→  每个采样点用多少 bit 表示            │
  │  Channel      ──→  有多少路独立的音频信号               │
  │  Format       ──→  采样点的编码方式 (S16LE/F32...)      │
  │  Frame        ──→  同一采样时刻所有 Channel 的数据合集  │
  │  Buffer Size  ──→  一次读写操作包含的 Frame 数          │
  │  Latency      ──→  从输入到输出的时间延迟               │
  └──────────────────────────────────────────────────────────┘
  
  它们之间的关系:
    Frame Size (bytes)  = Channel Count × Bytes per Sample
    Buffer Size (bytes) = Frame Count × Frame Size
    Buffer Duration (s) = Frame Count / Sample Rate
    Latency ≈ Buffer Duration × N (N = buffer 级数)
```

---

## 2. Sample Rate (采样率)

### 2.1 概念

采样率 = 每秒对模拟信号采集的次数，单位 Hz。

```
采样率直觉理解:

  48000 Hz = 每秒取 48000 个点来描述声波
  
  ──▶ 时间轴
  │·  ·  ·  ·  ·  ·  ·  ·│   ← 每个 "·" 是一个采样点
  │      1/48000 s        │   ← 两个点的间隔 ≈ 20.83 µs
```

### 2.2 代码中的 Sample Rate

```java
// === Android Java 层 ===
// 创建 AudioTrack 时指定采样率
AudioTrack track = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA)
        .build())
    .setAudioFormat(new AudioFormat.Builder()
        .setSampleRate(48000)          // ← 采样率
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
        .build())
    .setBufferSizeInBytes(bufferSize)
    .build();

// 获取设备原生采样率 (HAL 输出采样率)
int nativeSR = AudioTrack.getNativeOutputSampleRate(AudioManager.STREAM_MUSIC);
// 通常返回 48000 (Android 默认)
```

```c
// === ALSA/TinyALSA C 层 ===
struct pcm_config config = {
    .channels = 2,
    .rate = 48000,              // ← 采样率
    .period_size = 240,
    .period_count = 4,
    .format = PCM_FORMAT_S16_LE,
};
struct pcm *pcm = pcm_open(card, device, PCM_OUT, &config);
```

### 2.3 采样率不匹配会发生什么？

```
场景: App 输出 44100Hz, 但 HAL 要求 48000Hz

  AudioFlinger 自动插入 Resampler (多相 FIR 滤波器)
  44100 → 重采样 → 48000
  
  代价:
    - 额外 CPU 开销 (~1-3% per track)
    - 理论上引入微小失真 (但高质量 SRC 几乎不可闻)
    
  最佳实践:
    App 直接用 48000Hz 输出 → 避免重采样
    AudioTrack.getNativeOutputSampleRate() 获取原生率
```

---

## 3. Bit Depth / Format (位深 / 格式)

### 3.1 概念

位深 = 每个采样点的精度。Format = 位深 + 数据类型 + 字节序的组合。

```
常见格式对照:

  名称           位深   类型    范围                  Android 常量
  ───────────────────────────────────────────────────────────────────
  S16_LE         16-bit 有符号  [-32768, +32767]      ENCODING_PCM_16BIT
  S24_LE         24-bit 有符号  [-8388608, +8388607]  ENCODING_PCM_24BIT_PACKED
  S24_3LE        24-bit 3字节   同上                   —
  S32_LE         32-bit 有符号  [-2^31, +2^31-1]      ENCODING_PCM_32BIT
  FLOAT_LE       32-bit 浮点    [-1.0, +1.0] (标称)   ENCODING_PCM_FLOAT
  
  "S" = Signed (有符号)
  "LE" = Little-Endian (低字节在前, x86/ARM 默认)

内存中一个 S16_LE 采样:
  值 = 16384 (0x4000)
  内存: [0x00][0x40]   ← 低字节 0x00 在前
  
内存中一个 FLOAT_LE 采样:
  值 = 0.5f
  内存: [0x00][0x00][0x00][0x3F]  ← IEEE 754 表示
```

### 3.2 代码中的 Format

```java
// === Android Java 层 ===
AudioFormat format = new AudioFormat.Builder()
    .setEncoding(AudioFormat.ENCODING_PCM_FLOAT)  // ← 格式
    .setSampleRate(48000)
    .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
    .build();

// 获取每个采样的字节数
int bytesPerSample;
switch (encoding) {
    case AudioFormat.ENCODING_PCM_8BIT:    bytesPerSample = 1; break;
    case AudioFormat.ENCODING_PCM_16BIT:   bytesPerSample = 2; break;
    case AudioFormat.ENCODING_PCM_24BIT_PACKED: bytesPerSample = 3; break;
    case AudioFormat.ENCODING_PCM_32BIT:   bytesPerSample = 4; break;
    case AudioFormat.ENCODING_PCM_FLOAT:   bytesPerSample = 4; break;
}
```

```cpp
// === AudioFlinger 内部 ===
// AudioFlinger 混音器始终使用 float32 进行混音
// frameworks/av/services/audioflinger/AudioMixer.cpp
//
// 输入: 各 Track 可能是 S16/S32/Float
//   → 统一转换为 float32
//   → 混音 (float 相加)
//   → 输出到 HAL (转回 HAL 要求的格式)

// 格式转换示例:
// S16 → Float:
//   float_val = (float)s16_val / 32768.0f;
// Float → S16:
//   s16_val = (int16_t)(float_val * 32767.0f);
//   // 注意: 需要 clamp 到 [-32768, +32767] 防止溢出!
```

---

## 4. Channel (声道)

### 4.1 概念

Channel = 独立的音频信号通道。立体声 = 2 Channel (Left + Right)。

```
常见声道配置:

  Mono   (单声道):     1 ch   用途: 语音通话、麦克风录音
  Stereo (立体声):     2 ch   用途: 音乐、媒体播放
  2.1:                 3 ch   Stereo + LFE (低音炮)
  5.1 Surround:        6 ch   FL + FR + FC + LFE + SL + SR
  7.1 Surround:        8 ch   5.1 + BL + BR

  Android Channel Mask:
    CHANNEL_OUT_MONO    = 0x04         (Center 或 Front Left)
    CHANNEL_OUT_STEREO  = 0x0C         (Front Left + Front Right)
    CHANNEL_OUT_5POINT1 = 0xFC         (FL+FR+FC+LFE+BL+BR)
    
  ALSA Channel Map:
    Mono:    [FC]
    Stereo:  [FL, FR]
    5.1:     [FL, FR, FC, LFE, RL, RR]
```

### 4.2 代码中的 Channel

```java
// === Android Java 层 ===
// 播放
AudioFormat.CHANNEL_OUT_STEREO   // 2ch 播放
AudioFormat.CHANNEL_OUT_MONO     // 1ch 播放

// 录音
AudioFormat.CHANNEL_IN_MONO      // 1ch 录音 (单麦)
AudioFormat.CHANNEL_IN_STEREO    // 2ch 录音 (双麦)
```

```c
// === TinyALSA C 层 ===
struct pcm_config config = {
    .channels = 2,              // ← 声道数
    .rate = 48000,
    .period_size = 240,
    .period_count = 4,
    .format = PCM_FORMAT_S16_LE,
};

// 车载 TDM 多声道:
struct pcm_config tdm_config = {
    .channels = 8,              // ← 8 声道 TDM
    .rate = 48000,
    .period_size = 480,
    .period_count = 4,
    .format = PCM_FORMAT_S32_LE,
};
```

---

## 5. Frame (帧)

### 5.1 概念

**Frame = 同一采样时刻, 所有 Channel 的采样值合在一起。** 这是音频软件中最核心的计量单位。

```
Frame 的直觉理解:

  时间轴 →    t0        t1        t2        t3
             ┌──┐      ┌──┐      ┌──┐      ┌──┐
  Channel L: │L0│      │L1│      │L2│      │L3│
             └──┘      └──┘      └──┘      └──┘
             ┌──┐      ┌──┐      ┌──┐      ┌──┐
  Channel R: │R0│      │R1│      │R2│      │R3│
             └──┘      └──┘      └──┘      └──┘
             
             ├Frame0──┤├Frame1──┤├Frame2──┤├Frame3──┤
             
  每个 Frame 包含: 1 个 L 采样 + 1 个 R 采样

  Frame Size (字节):
    S16 Mono:    1 × 2 = 2 bytes/frame
    S16 Stereo:  2 × 2 = 4 bytes/frame
    Float Stereo: 2 × 4 = 8 bytes/frame
    S32 8ch:     8 × 4 = 32 bytes/frame
```

### 5.2 Frame vs Sample 的区别（最常见的混淆）

```
⚠️ 关键区分:

  Sample = 单个声道的一个采样值
  Frame  = 所有声道在同一时刻的采样值集合
  
  S16 Stereo 的 1 个 Frame:
    [L_lo][L_hi][R_lo][R_hi]    ← 4 bytes
    └─Sample─┘  └─Sample─┘
    └───────── Frame ─────────┘
    
  Frame Count = 1000:
    包含 1000 × 2 = 2000 个 Sample
    占用 1000 × 4 = 4000 bytes
    时长 = 1000 / 48000 = 20.83 ms

  Android API 中的命名:
    AudioTrack.getBufferSizeInFrames()  → 返回 Frame 数
    AudioTrack.write(short[], ...)       → 参数是 Sample 数!
    
    ⚠️ Stereo 时: sampleCount = frameCount × 2
    这是初学者最容易踩的坑!
```

### 5.3 代码中的 Frame

```java
// === Android Java 层 ===
// 获取最小 buffer 大小 (单位: bytes)
int minBufSize = AudioTrack.getMinBufferSize(
    48000,                               // sampleRate
    AudioFormat.CHANNEL_OUT_STEREO,      // channelMask  
    AudioFormat.ENCODING_PCM_16BIT       // format
);
// 返回值单位是 bytes, 不是 frames!

// bytes → frames 的换算:
int channelCount = 2;  // stereo
int bytesPerSample = 2; // 16-bit
int frameSize = channelCount * bytesPerSample; // = 4 bytes/frame
int frameCount = minBufSize / frameSize;

// write() 的参数陷阱:
short[] data = new short[frameCount * channelCount]; // Sample 数!
track.write(data, 0, data.length);  // ← 传入的是 Sample 数, 不是 Frame 数!

// 而 float 版本:
float[] fdata = new float[frameCount * channelCount];
track.write(fdata, 0, fdata.length, AudioTrack.WRITE_BLOCKING);
```

```cpp
// === AudioFlinger C++ 层 ===
// frameworks/av/services/audioflinger/Threads.cpp
//
// PlaybackThread 核心变量:
//   mFrameCount   = HAL buffer 的 Frame 数 (如 960)
//   mSampleRate   = HAL 采样率 (如 48000)
//   mChannelCount = HAL 声道数 (如 2)
//   mFrameSize    = mChannelCount * bytesPerSample (如 4)
//
// 处理循环:
//   每次 threadLoop() 处理 mFrameCount 个 frames
//   耗时 = mFrameCount / mSampleRate = 960/48000 = 20ms
```

---

## 6. Buffer Size / Frame Count (缓冲区大小)

### 6.1 概念

Buffer Size = 音频缓冲区能容纳的 Frame 数。它直接决定了**延迟 (Latency)** 和**稳定性 (Glitch-free)** 之间的权衡。

```
Buffer 与延迟的关系:

  Buffer 大 → 延迟高, 但不容易 underrun (卡顿)
  Buffer 小 → 延迟低, 但 CPU 调度稍慢就 underrun
  
  ┌─────────────────────────────────────────┐
  │              Ring Buffer                │
  │  ┌────┬────┬────┬────┬────┬────┬────┐  │
  │  │ F0 │ F1 │ F2 │ F3 │ F4 │ F5 │ F6 │  │
  │  └────┴────┴────┴────┴────┴────┴────┘  │
  │       ↑ Read (HAL 消费)                 │
  │                          ↑ Write (App 生产)
  │                                         │
  │  Underrun: Write 跟不上 Read → 空了!    │
  │  Overrun:  Read 跟不上 Write → 满了!    │
  └─────────────────────────────────────────┘
  
  典型配置:
    Normal Track:   Buffer = 4 × period = 4 × 240 = 960 frames = 20ms
    Fast Track:     Buffer = 2 × period = 2 × 128 = 256 frames ≈ 5ms
    AAudio MMAP:    Buffer = 1-2 × period ≈ 128 frames ≈ 2.67ms
    Deep Buffer:    Buffer = 4 × 1920 = 7680 frames = 160ms (省电)
```

### 6.2 Period / Period Count

```
Buffer 的内部组织:

  Buffer = Period Count × Period Size (frames)
  
  Period (周期) = HAL 一次 DMA 传输的数据量
    - DMA 每传完一个 Period → 触发一次中断
    - HAL 在中断回调中填入下一个 Period 的数据
    
  示例: period_size=240, period_count=4
  
    ┌─────────┬─────────┬─────────┬─────────┐
    │Period 0 │Period 1 │Period 2 │Period 3 │
    │240 frame│240 frame│240 frame│240 frame│
    └─────────┴─────────┴─────────┴─────────┘
    │◄──── Buffer = 960 frames = 20ms ────►│
    
    DMA 正在送出 Period 0 的数据给 DAC
    同时 CPU 在填写 Period 2 或 3 → 双缓冲/多缓冲

  ALSA 中:
    period_size  = 一个周期的 Frame 数
    period_count = 周期个数 (= buffer_size / period_size)
    buffer_size  = 总 Frame 数
```

### 6.3 代码中的 Buffer Size

```java
// === Android Java 层 ===
// 获取最小 buffer (bytes)
int minBuf = AudioTrack.getMinBufferSize(48000,
    AudioFormat.CHANNEL_OUT_STEREO,
    AudioFormat.ENCODING_PCM_16BIT);
// minBuf ≈ 3840 bytes = 960 frames × 4 bytes/frame

// 创建 AudioTrack 时可以指定更大的 buffer
AudioTrack track = new AudioTrack.Builder()
    .setBufferSizeInBytes(minBuf * 2)  // 2倍最小值, 更安全
    .build();

// 运行时动态调整 buffer (Android 7.0+)
track.setBufferSizeInFrames(960);  // 动态调小 → 降低延迟
int actualBuf = track.getBufferSizeInFrames(); // 实际生效值
```

```c
// === TinyALSA 层 ===
struct pcm_config config;
config.period_size = 240;     // 每个 period 240 frames (5ms @ 48kHz)
config.period_count = 4;      // 4 个 period
// buffer_size = 240 × 4 = 960 frames = 20ms

struct pcm *pcm = pcm_open(0, 0, PCM_OUT, &config);

// 一次 write 写入一个 period 的数据
int frame_size = config.channels * 2; // S16 stereo = 4 bytes
pcm_writei(pcm, buffer, config.period_size);
// 写入 240 frames × 4 bytes = 960 bytes
```

---

## 7. Latency (延迟)

### 7.1 延迟的组成

```
端到端音频延迟拆解:

  App write() → ... → 扬声器出声

  ┌────────────────────────────────────────────────────────┐
  │ 1. App Buffer        写入但尚未消费的数据              │
  │    ~0-20ms           取决于 App 提前写入量             │
  ├────────────────────────────────────────────────────────┤
  │ 2. AudioFlinger      Server buffer + 混音处理          │
  │    ~5-20ms           Normal: 20ms / Fast: 5ms          │
  ├────────────────────────────────────────────────────────┤
  │ 3. HAL Buffer        ALSA ring buffer 中排队           │
  │    ~5-20ms           period_size × (period_count-1)    │
  ├────────────────────────────────────────────────────────┤
  │ 4. DSP 处理          AudioReach Graph 处理延迟         │
  │    ~1-5ms            取决于 Graph 中模块数量           │
  ├────────────────────────────────────────────────────────┤
  │ 5. DAC 转换          数字→模拟转换                     │
  │    ~0.1-1ms          取决于 DAC 芯片                   │
  ├────────────────────────────────────────────────────────┤
  │ 6. 功放+扬声器       物理响应时间                      │
  │    ~0.1-0.5ms                                          │
  └────────────────────────────────────────────────────────┘
  
  总延迟 (典型值):
    Normal Path:   50-100ms
    Fast Path:     20-40ms
    AAudio MMAP:   10-20ms
    Offload Path:  100-300ms (大 buffer 换低功耗)
```

### 7.2 代码中获取延迟

```java
// === Android Java 层 ===
// 获取输出延迟
AudioManager am = (AudioManager) getSystemService(AUDIO_SERVICE);

// 方法 1: 获取 HAL 报告的延迟
String latencyStr = am.getProperty(AudioManager.PROPERTY_OUTPUT_LATENCY);

// 方法 2: AudioTrack 直接获取
AudioTrack track = ...;
int latencyMs = track.getLatency(); // 非公开 API, 反射调用
// 或
int bufFrames = track.getBufferSizeInFrames();
float bufLatencyMs = (float) bufFrames / track.getSampleRate() * 1000;

// === AAudio (最低延迟路径) ===
// AAudio 通过 PerformanceMode 控制延迟
AAudioStreamBuilder_setPerformanceMode(builder,
    AAUDIO_PERFORMANCE_MODE_LOW_LATENCY);

// 获取实际延迟
int32_t framesPerBurst;
AAudioStream_getFramesPerBurst(stream, &framesPerBurst);
// framesPerBurst ≈ 48-192 frames → 1-4ms 级别延迟
```

---

## 8. 完整公式速查表

```
核心公式:
  ──────────────────────────────────────────────────────────
  Frame Size (bytes)  = Channel Count × Bytes per Sample
  Buffer Size (bytes) = Frame Count × Frame Size
  Buffer Duration (s) = Frame Count / Sample Rate
  Bitrate (bps)       = Sample Rate × Bit Depth × Channels
  BCLK (Hz)           = Sample Rate × Bit Depth × Channels
  MCLK (Hz)           = Sample Rate × 256 (或 512)
  
  换算示例 (S16 Stereo 48kHz, 960 frames):
  ──────────────────────────────────────────────────────────
  Frame Size   = 2 × 2          = 4 bytes
  Buffer Size  = 960 × 4        = 3840 bytes ≈ 3.75 KB
  Duration     = 960 / 48000    = 0.02 s = 20 ms
  Bitrate      = 48000 × 16 × 2 = 1,536,000 bps ≈ 1.5 Mbps
  
  换算示例 (Float 8ch 48kHz, 480 frames):
  ──────────────────────────────────────────────────────────
  Frame Size   = 8 × 4          = 32 bytes
  Buffer Size  = 480 × 32       = 15,360 bytes = 15 KB
  Duration     = 480 / 48000    = 0.01 s = 10 ms
  Bitrate      = 48000 × 32 × 8 = 12,288,000 bps ≈ 12.3 Mbps
```

---

## 9. 概念关系全景图

```mermaid
graph TD
    SR["Sample Rate<br/>48000 Hz"]
    BD["Bit Depth / Format<br/>S16_LE = 2 bytes"]
    CH["Channel Count<br/>Stereo = 2"]
    
    SR --> FRAME["Frame<br/>=所有 CH 在一个采样点的数据<br/>= 2 × 2 = 4 bytes"]
    BD --> FRAME
    CH --> FRAME
    
    FRAME --> PERIOD["Period<br/>= 240 frames<br/>= 240 × 4 = 960 bytes<br/>= 5 ms"]
    
    PERIOD --> BUFFER["Buffer<br/>= 4 periods<br/>= 960 frames<br/>= 3840 bytes<br/>= 20 ms"]
    
    BUFFER --> LATENCY["Latency<br/>≈ Buffer + DSP + DAC<br/>≈ 20-100 ms"]
    
    SR --> BITRATE["Bitrate<br/>= 48000 × 16 × 2<br/>= 1536 kbps"]
    BD --> BITRATE
    CH --> BITRATE
```

---

## 10. 常见陷阱 FAQ

| # | 陷阱 | 说明 |
|:---|:---|:---|
| 1 | **Frame 和 Sample 混淆** | `AudioTrack.write(short[])` 的 length 是 **Sample 数**, 不是 Frame 数。Stereo 时 sampleCount = frameCount × 2 |
| 2 | **getMinBufferSize 返回 bytes** | 不是 frames! 需要除以 frameSize 才是 frame count |
| 3 | **Buffer 越大≠越好** | Buffer 大 → 延迟高 → 游戏/通话体验差 |
| 4 | **44100 vs 48000** | Android 默认 48kHz, App 如果用 44100 会触发重采样 |
| 5 | **Float 不会溢出?** | Float 可以超过 1.0, 但 DAC 只能输出 [-1.0, 1.0], 超出即削波失真 |
| 6 | **8-bit PCM 是 unsigned** | 大多数平台 8-bit PCM 是 unsigned (128=静音), 其他位深是 signed |
| 7 | **Interleaved 顺序** | LRLRLR... 是标准 interleaved; 别假设是 LLLLRRRR |

---

## 11. 关键参考 (References)

1.  [Android AudioTrack API Reference](https://developer.android.com/reference/android/media/AudioTrack)
2.  [AAudio API Reference](https://developer.android.com/ndk/guides/audio/aaudio/aaudio)
3.  [ALSA PCM Interface](https://www.alsa-project.org/alsa-doc/alsa-lib/pcm.html)
4.  [TinyALSA GitHub](https://github.com/tinyalsa/tinyalsa)
5.  [Android Audio Latency - Official](https://source.android.com/docs/core/audio/latency)

---
*返回：[数字音频基础](./03-Digital-Audio-Fundamentals.md) | [Android 音频概览](../04-Android-Audio-Stack/01-Overview.md)*
