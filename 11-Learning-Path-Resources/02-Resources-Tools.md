# 音频开发资源与工具推荐 (Resources & Tools)

好的工具和参考资料可以事半功倍。以下按类别整理了音频行业最常用的经典书籍、开源项目、软件工具、在线资源和技术规范。

---

## 1. 经典书籍

### 1.1 声学与心理声学

| 书名 | 作者 | 核心内容 | 难度 |
|:---|:---|:---|:---|
| *The Master Handbook of Acoustics* | F. Alton Everest | 声学基础到录音室设计 | 入门→进阶 |
| *Fundamentals of Acoustics* | Lawrence Kinsler | 声学理论、波动方程 | 进阶 |
| *Psychoacoustics: Facts and Models* | Hugo Fastl | 心理声学实验与模型 | 专业 |
| *An Introduction to the Psychology of Hearing* | Brian Moore | 人耳听觉机制 | 进阶 |

### 1.2 数字音频与信号处理

| 书名 | 作者 | 核心内容 | 难度 |
|:---|:---|:---|:---|
| *Principles of Digital Audio* | Ken C. Pohlmann | 采样/量化/编码基础 | 入门 |
| *Understanding Digital Signal Processing* | Richard Lyons | FFT/滤波器/实际应用 | 入门→进阶 |
| *Digital Audio Signal Processing* | Udo Zölzer | EQ/混响/动态处理算法 | 进阶→专业 |
| *Adaptive Filter Theory* | Simon Haykin | AEC/自适应滤波器 | 专业 |
| *Discrete-Time Signal Processing* | Oppenheim & Schafer | DSP 理论经典 | 专业 |
| *DAFX: Digital Audio Effects* | Udo Zölzer (编) | 音效算法实现集合 | 进阶 |

### 1.3 系统工程与驱动

| 书名 | 作者 | 核心内容 | 难度 |
|:---|:---|:---|:---|
| *Embedded Android* | Karim Yaghmour | Android HAL/驱动 | 进阶 |
| *Linux Device Drivers* | Jonathan Corbet | Linux 驱动开发 | 进阶 |
| *深入理解 Android* | 邓凡平 | Android 框架原理 | 进阶 |
| *Professional Linux Kernel Architecture* | Wolfgang Mauerer | Linux 内核架构 | 专业 |

### 1.4 专业方向

| 方向 | 推荐书籍 |
|:---|:---|
| **蓝牙音频** | *Bluetooth Essentials for Programmers* - Albert Huang |
| **车载音频** | *Automotive Audio Signal Processing* - H. Brandenstein |
| **声学设计** | *Sound System Engineering* - Don & Carolyn Davis |
| **语音处理** | *Speech and Audio Signal Processing* - Ben Gold |
| **空间音频** | *3D Audio* - Agnieszka Rogińska |

---

## 2. 开源项目

### 2.1 核心框架与库

| 项目 | 语言 | 用途 | 链接 |
|:---|:---|:---|:---|
| **WebRTC** | C++ | 3A 算法 (AEC/NS/AGC)、音频传输 | [webrtc.org](https://webrtc.org/) |
| **Oboe** | C++ | Android 低延迟音频 (封装 AAudio) | [github.com/google/oboe](https://github.com/google/oboe) |
| **FFmpeg** | C | 音视频编解码/转码/协议 | [ffmpeg.org](https://ffmpeg.org/) |
| **TinyALSA** | C | Android ALSA 用户态轻量库 | [github.com/tinyalsa/tinyalsa](https://github.com/tinyalsa/tinyalsa) |
| **PipeWire** | C | 下一代 Linux 音视频框架 | [pipewire.org](https://pipewire.org/) |
| **PulseAudio** | C | Linux 音频服务器 (传统) | [freedesktop.org/wiki/Software/PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/) |
| **JACK** | C | 专业低延迟音频服务器 | [jackaudio.org](https://jackaudio.org/) |

### 2.2 算法与工具

| 项目 | 用途 | 链接 |
|:---|:---|:---|
| **SpeexDSP** | 回声消除/降噪/重采样 | [speex.org](https://www.speex.org/) |
| **RNNoise** | 基于 RNN 的降噪 | [github.com/xiph/rnnoise](https://github.com/xiph/rnnoise) |
| **Opus** | 低延迟音频编解码 | [opus-codec.org](https://opus-codec.org/) |
| **Sox** | 命令行音频处理 ("音频瑞士军刀") | [sox.sourceforge.net](http://sox.sourceforge.net/) |
| **librosa** | Python 音频分析 (MIR) | [librosa.org](https://librosa.org/) |
| **pytorch-audio (torchaudio)** | PyTorch 音频处理 | [pytorch.org/audio](https://pytorch.org/audio/) |
| **PESQ** | 语音质量客观评估 (Python) | [github.com/ludlows/PESQ](https://github.com/ludlows/PESQ) |

---

## 3. 软件工具

### 3.1 音频分析与编辑

| 工具 | 平台 | 费用 | 核心功能 |
|:---|:---|:---|:---|
| **Audacity** | Win/Mac/Linux | 免费 | 波形编辑、频谱图、基础音效 |
| **Adobe Audition** | Win/Mac | 收费 | 专业降噪、多轨编辑 |
| **Sonic Visualiser** | Win/Mac/Linux | 免费 | 学术级频谱/音高分析 |
| **Reaper** | Win/Mac/Linux | $60 | DAW、VST 插件开发调试 |

### 3.2 声学测量

| 工具 | 费用 | 用途 |
|:---|:---|:---|
| **REW (Room EQ Wizard)** | 免费 | 房间声学分析、频响测量、EQ 调试 |
| **Audio Precision APx500** | 收费 (硬件配套) | 行业标准：THD+N/SNR/频响自动化测试 |
| **ARTA** | 免费/收费 | 扬声器/麦克风频响测量 |
| **SMAART** | 收费 | 实时双通道声学分析 |

### 3.3 Android 调试工具链

```bash
# 音频流状态
adb shell dumpsys media.audio_flinger
adb shell dumpsys media.audio_policy
adb shell dumpsys audio

# PCM 录制/播放
adb shell tinycap /sdcard/test.wav -D 0 -d 0 -c 2 -r 48000 -b 16
adb shell tinyplay /sdcard/test.wav -D 0 -d 0

# Mixer 控件
adb shell tinymix -D 0
adb shell tinymix 'Volume' '80'

# 性能分析
adb shell perfetto --txt -c /data/misc/perfetto-configs/audio.cfg -o /data/misc/perfetto-traces/audio.perfetto-trace

# 音频 dump
adb shell setprop vendor.audio.hal.dump 1   # 使能 HAL 侧 dump
adb pull /data/vendor/audio/                 # 拉取 dump 文件
```

### 3.4 MATLAB / Python 常用脚本

```python
# Python 音频快速分析 (常用库)
import numpy as np
import scipy.signal as signal
import soundfile as sf
import matplotlib.pyplot as plt

# 读取 WAV
data, fs = sf.read('test.wav')

# 频谱分析
f, Pxx = signal.welch(data, fs, nperseg=4096)
plt.semilogx(f, 10 * np.log10(Pxx))
plt.xlabel('Frequency (Hz)')
plt.ylabel('PSD (dB/Hz)')
plt.show()

# THD 计算 (基频 + 谐波)
fft_data = np.fft.rfft(data)
freqs = np.fft.rfftfreq(len(data), 1/fs)
# ... 提取基频和谐波分量计算 THD
```

---

## 4. 在线资源

### 4.1 官方文档

| 资源 | 内容 | 链接 |
|:---|:---|:---|
| **AOSP Audio** | Android 音频架构/API | [source.android.com/audio](https://source.android.com/devices/audio) |
| **ALSA Project** | Linux ALSA 驱动文档 | [alsa-project.org](https://www.alsa-project.org/) |
| **Bluetooth SIG** | BT 协议规范 | [bluetooth.com/specifications](https://www.bluetooth.com/specifications/) |
| **Qualcomm Developer** | 高通音频 SDK | [developer.qualcomm.com](https://developer.qualcomm.com/) |
| **ADI A2B** | A2B 技术文档 | [analog.com/a2b](https://www.analog.com/en/applications/technology/a2b-audio-bus.html) |

### 4.2 学术与社区

| 资源 | 定位 | 链接 |
|:---|:---|:---|
| **Hydrogenaudio** | 极客级音频技术讨论 | [hydrogenaudio.org](https://www.hydrogenaudio.org/) |
| **KVR Audio** | 音效插件/音乐制作 | [kvraudio.com](https://www.kvraudio.com/) |
| **DSPRelated** | DSP 算法教程/博客 | [dsprelated.com](https://www.dsprelated.com/) |
| **Stack Overflow** | Android 音频 Q&A | [stackoverflow.com/tagged/android-audio](https://stackoverflow.com/questions/tagged/android-audio) |
| **Stanford CCRMA** | 音频/音乐计算研究 | [ccrma.stanford.edu](https://ccrma.stanford.edu/) |
| **Xiph.org** | 开源音频编解码标准 | [xiph.org](https://xiph.org/) |

---

## 5. 技术规范与标准

| 标准 | 组织 | 内容 |
|:---|:---|:---|
| **ITU-T P.862 (PESQ)** | ITU | 窄带/宽带语音质量评估 |
| **ITU-T P.863 (POLQA)** | ITU | 全频带语音质量评估 |
| **ITU-R BS.1770** | ITU | 响度测量标准 (LUFS) |
| **EBU R128** | EBU | 广播响度归一化 |
| **IEC 61672** | IEC | 声级计标准 |
| **ISO 226:2003** | ISO | 等响曲线 |
| **AES67** | AES | AoIP 音频互操作 |
| **A2B (AD2428W)** | ADI | 汽车音频总线 |
| **Bluetooth A2DP/LE Audio** | Bluetooth SIG | 蓝牙音频传输 |

---

## 6. 技术大会与课程

| 名称 | 类型 | 方向 | 链接 |
|:---|:---|:---|:---|
| **AES Convention** | 大会 | 音频工程全方向 | [aes.org](https://www.aes.org/) |
| **Android Dev Summit** | 大会 | Android 音频 API | [developer.android.com](https://developer.android.com/) |
| **Embedded Linux Conference** | 大会 | Linux/ALSA 驱动 | [events.linuxfoundation.org](https://events.linuxfoundation.org/) |
| **Bluetooth World** | 大会 | BT/LE Audio | [bluetooth.com](https://www.bluetooth.com/) |
| **Coursera - Audio Signal Processing** | 课程 | DSP 基础 | Stanford/UPF 联合课程 |
| **Coursera - Music & Audio** | 课程 | 音乐信息检索 | Georgia Tech |
| **MIT OCW 6.341** | 课程 | 离散信号处理 | [ocw.mit.edu](https://ocw.mit.edu/) |
| **Qualcomm 合作伙伴培训** | 认证 | 高通音频平台 | 需合作伙伴资质 |

---
[返回主目录](../README.md)
