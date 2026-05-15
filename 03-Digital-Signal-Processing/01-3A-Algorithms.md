# 语音通信 3A 算法 (AEC, ANS, AGC)

在语音通话和会议系统中，3A 算法是保证通话质量的核心。它们分别是：回声消除 (AEC)、自动降噪 (ANS) 和 自动增益控制 (AGC)。

---

## 1. 回声消除 (AEC, Acoustic Echo Cancellation)

### 1.1 产生原因
当扬声器播放的声音再次进入麦克风，并传回给对方时，对方就会听到自己的回声。

### 1.2 原理
AEC 的核心是**自适应滤波器 (Adaptive Filter)**。它利用“远端参考信号”来模拟回声路径，从而从麦克风采集的混合信号中减去估计的回声。

\`\`\`mermaid
graph LR
    FarIn[远端参考信号 x] --> SPK[扬声器]
    SPK -- 回声路径 h --> MIC[麦克风]
    Voice[近端语音 s] --> MIC
    MIC -- 混合信号 d = s + y --> Sub[减法器]
    
    FarIn --> Filter[自适应滤波器 h_hat]
    Filter -- 估计回声 y_hat --> Sub
    Sub -- 误差信号 e = s + y - y_hat --> Out[输出 / 回传远端]
    Out --> Filter
\`\`\`

*   **常用算法**：NLMS (归一化最小均方)、FDAF (频域自适应滤波)。
*   **挑战**：双讲 (Double-talk) 检测、非线性失真。

---

## 2. 自动降噪 (ANS, Automatic Noise Suppression)

### 2.1 目标
消除环境中的稳态噪声（如风扇声、空调声）和瞬态噪声，提取清晰的人声。

### 2.2 常用方法
*   **谱减法 (Spectral Subtraction)**：假设噪声是平稳的，从混合信号的频谱中减去噪声估计谱。
*   **维纳滤波 (Wiener Filtering)**：根据最小均方误差准则，在频域对信号进行加权增益调整。
*   **深度学习降噪 (AI-NS)**：利用 RNN 或 Transformer 模型，在极端嘈杂环境下提取人声。

---

## 3. 自动增益控制 (AGC, Automatic Gain Control)

### 3.1 目标
无论说话人离麦克风远近，或者说话声音大小，都能将输出音量维持在一个人耳舒适的恒定范围内。

### 3.2 核心逻辑
*   **能量检测**：实时计算信号的 RMS 或峰值能量。
*   **增益映射**：根据设定的目标电平 (Target Level)，计算所需的补偿增益。
*   **动态控制**：包含 **启动时间 (Attack Time)** 和 **释放时间 (Release Time)**，防止音量突变带来的听感不适。

---

## 4. 处理顺序 (Processing Order)

在典型的音频 Pipeline 中，3A 的顺序通常为：
**麦克风采集 -> AEC -> ANS -> AGC -> 编码传输**

*注：AEC 必须放在最前面，因为它需要原始的参考信号。*

---

## 5. 关键参考 (References)

1.  *Adaptive Filter Theory* - Simon Haykin
2.  [Acoustic Echo Cancellation - Wikipedia](https://en.wikipedia.org/wiki/Echo_suppression_and_cancellation)
3.  [WebRTC Audio Processing Library](https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_processing/)

---
*Next Topic: [音效处理：EQ, DRC 与 空间音频](./02-Audio-Effects.md)*
