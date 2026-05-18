# 车载音频系统概览 (Automotive Audio System Overview)

车载音频系统已从单纯的“娱乐系统”演变为一个集成安全警报、语音交互、多音区娱乐及主动降噪的复杂分布式系统。

---

## 1. 以流为中心的架构 (Stream-Centric Architecture)

在 AAOS 中，音频被抽象为“逻辑流”与“物理流”。

*   **逻辑流 (Logical Streams)**：由 App 发出，带有 `AudioAttributes` 标签（如：Music, Navigation）。
*   **物理流 (Physical Streams)**：由 `AudioFlinger` 混音后输出到 HAL 层的特定总线 (Bus)。一旦进入物理流，原始的 Context 信息将丢失。

### 1.1 外部音源直通 (External Sounds)
安全音（Chimes）和警告音（Warnings）通常**不经过 Android 路由**。
*   **原因**：为了通过安全认证（ASIL）并保证极低延迟。
*   **实现**：由外部 MCU 直接通过 I2S 注入到 DSP 功放，或通过 HAL 层的 `createAudioPatch` 建立硬件直通。

---

## 2. 仲裁矩阵与优先级 (Arbitration Matrix)

| 优先级 | 类型 (Context) | 典型示例 | 处理策略 |
| :--- | :--- | :--- | :--- |
| **P0 (最高)** | EMERGENCY | 气囊弹出警告、紧急求助 | 硬件直通，强制静音一切 |
| **P1** | SAFETY | 倒车雷达、车道偏离预警 | 压低背景音 (Ducking) |
| **P2** | VEHICLE_STATUS | 低压警告、车门未关 | 并发播放或淡入淡出 |
| **P3** | ANNOUNCEMENT | 交通广播、系统通知 | 抢占焦点 |
| **P4 (最低)** | MUSIC / MEDIA | 音乐、视频、游戏 | 被随时中断 |

---

## 3. 车载专用功能

### 3.1 车内通信 (ICC, In-Car Communication)
解决驾驶员与后排乘客交流困难的问题。
*   **原理**：利用麦克风阵列拾取前排声音，经过算法处理（回声消除、反馈抑制）后，通过后排扬声器实时播放。
*   **延迟要求**：通常要求系统端到端延迟小于 **10ms**。

### 3.2 语音 UI 与 Snapdragon Voice Activation (SVA)
*   **Snapdragon Voice Activation**：高通特有的低功耗语音唤醒方案，运行在 **LPASS (Low Power Audio Subsystem)** 中。
*   **多音区识别**：支持识别唤醒词来自哪个座位，从而定向开启对应座位的语音交互。

---

## 4. 关键参考 (References)

1.  [Qualcomm: SA8155/SA8295 Android Audio Overview](https://developer.qualcomm.com/)
2.  [Android Automotive Audio Official Guide](https://source.android.com/devices/automotive/audio)
