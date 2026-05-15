# 音频客观测试指标 (Objective Audio Metrics)

客观测试通过标准化的仪器测量，为音频系统的性能提供可量化的评估数据。这些指标排除了主观听感的误差，是音频产品生产和研发的基石。

---

## 1. 核心性能指标

### 1.1 总谐波失真加噪声 (THD+N)
*   **定义**：输出信号中谐波成分以及噪声与原始信号的比值。
*   **意义**：数值越低，代表系统对原始声音的还原度越高。0.1% 通常被认为是高保真 (Hi-Fi) 的门槛。

### 1.2 信噪比 (SNR, Signal-to-Noise Ratio)
*   **定义**：有效信号功率与噪声功率的比值，单位为 dB。
*   **意义**：SNR 越高，背景噪声越干净。CD 的理论 SNR 为 96dB。

### 1.3 频率响应 (Frequency Response)
*   **定义**：系统在不同频率下的增益变化。
*   **理想状态**：在 20Hz - 20kHz 范围内呈一条平直的直线（±3dB 甚至更小）。

### 1.4 动态范围 (Dynamic Range)
*   **定义**：系统能处理的最大不失真信号与最小可察觉信号（底噪）之间的差值。
*   **意义**：对于再现音乐中的细节起伏至关重要。

---

## 2. 测试系统拓扑

典型的音频客观测试需要专业的音频分析仪（如 Audio Precision）。

```mermaid
graph LR
    Generator[音频分析仪 - 信号发生器] --> DUT[被测设备 - Device Under Test]
    DUT --> Analyzer[音频分析仪 - 分析接收端]
    Analyzer --> Computer[分析软件 - 绘制曲线]
```

---

## 3. 常见的专业测试仪器

*   **Audio Precision (AP)**：行业公认的“黄金标准”，用于高精度模拟和数字音频测试。
*   **B&K (Brüel & Kjær)**：专注于声学传感器、麦克风及人头录音系统的测量。
*   **Listen Inc. SoundCheck**：广泛用于产线自动化测试。

---

## 4. 客观与主观的权衡

虽然客观指标非常重要，但它们不能完全代表“好听”。
*   例如：电子管放大器的 THD 往往较高，但由于其产生的偶次谐波丰富，听感反而被认为更加“温暖”。
*   **建议**：客观指标用于找 Bug 和质量控制，主观评价用于音色调校。

---

## 5. 关键参考 (References)

1.  *Principles of Digital Audio* - Ken C. Pohlmann
2.  [Audio Precision: Fundamental of Audio Test](https://www.ap.com/technical-library/)
3.  [ITU-R BS.1770: Loudness Measurement Standard](https://www.itu.int/rec/R-REC-BS.1770)

---
*Next Topic: [行业通信标准与认证 (Industry Standards)](./02-Industry-Standards.md)*
