# 05. Linux 音频子系统 (Linux Audio Subsystem)

本模块介绍 Linux 内核及桌面系统的音频架构，这是 Android 底层驱动及嵌入式音频系统的技术支撑。

## 📖 章节导航

1.  **[ALSA 核心架构 (ALSA Architecture)](./01-ALSA-Architecture.md)**
    *   ALSA 系统分层：alsa-lib, 内核中间层与驱动。
    *   设备节点解析：pcm, control, timer。
    *   alsa-lib 与 tinyalsa 的区别与应用场景。

2.  **[ASoC 驱动模型 (ASoC Driver Model)](./02-ASoC-Driver-Model.md)**
    *   三大核心组件：Codec, Platform, Machine 驱动。
    *   DAPM (动态音频电源管理) 自动开关原理。
    *   DAI (数字音频接口) 标准化。

3.  **[现代 Linux 音频服务 (PulseAudio & PipeWire)](./03-PulseAudio-PipeWire.md)**
    *   PulseAudio：桌面音频混音与网络透明性。
    *   PipeWire：下一代统一音视频处理标准。
    *   低延迟图模型架构与 Jack 兼容性。

---
[返回主目录](../README.md)
