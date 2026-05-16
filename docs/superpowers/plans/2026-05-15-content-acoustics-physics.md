# Sound Wave Physics Content Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a high-quality, professional technical document `01-Sound-Wave-Physics.md` in the `01-Acoustics-Digital-Audio` directory.

**Architecture:** Following the "Classic Technical Document Structure": Definition -> Principles -> Diagrams (Mermaid) -> Formula/Code -> References.

**Tech Stack:** Markdown, Mermaid.

---

### Task 1: Write 01-Sound-Wave-Physics.md

**Files:**
- Create: `01-Acoustics-Digital-Audio/01-Sound-Wave-Physics.md`

- [ ] **Step 1: Write the content of 01-Sound-Wave-Physics.md**

Write the following content to `01-Acoustics-Digital-Audio/01-Sound-Wave-Physics.md`:

```markdown
# 声波物理特性 (Sound Wave Physics)

声音是音频工程的逻辑起点。理解声音如何产生、传播以及其物理参数，是掌握后续数字信号处理 (DSP) 和声学设计的前提。

---

## 1. 什么是声音 (Definition of Sound)

**声音 (Sound)** 是由物体振动产生的**机械波 (Mechanical Wave)**。
*   **传播介质**：声音必须依靠介质（固体、液体、气体）传播，真空中无法传声。
*   **波的类型**：在空气中，声音以**纵波 (Longitudinal Wave)** 的形式传播，即介质质点的振动方向与波的传播方向平行，形成交替的**疏部 (Rarefaction)** 和 **密部 (Compression)**。

---

## 2. 核心物理参数 (Core Physical Parameters)

### 2.1 频率 (Frequency, $f$) 与 周期 (Period, $T$)
*   **定义**：物体每秒振动的次数称为频率，单位为 **赫兹 (Hz)**。振动一次所需的时间称为周期。
*   **关系**：$f = 1 / T$
*   **听觉范围**：人耳能听到的频率范围约为 **20Hz - 20,000Hz (20kHz)**。
    *   < 20Hz: 次声波 (Infrasound)
    *   > 20kHz: 超声波 (Ultrasound)

### 2.2 波长 (Wavelength, $\lambda$)
*   **定义**：声波在传播方向上，相邻两个相同相位点（如两个波峰）之间的距离。
*   **计算公式**：
    $$\lambda = \frac{v}{f}$$
    *(其中 $v$ 为声速)*
*   **重要性**：波长决定了声音的衍射（绕射）能力。低频声音波长长（如 100Hz 约 3.4米），容易绕过障碍物；高频声音波长短，方向性更强。

### 2.3 声速 (Speed of Sound, $v$)
*   声速取决于介质的弹性系数和密度。
*   **空气中的声速**：在 20°C (68°F) 的干燥空气中，声速约为 **343 m/s**。
*   **温度影响**：温度越高，声速越快。近似公式：$v \approx 331.5 + 0.6T$ (其中 $T$ 为摄氏度)。

### 2.4 相位 (Phase, $\phi$)
*   **定义**：描述声波在特定时间点处于振动周期中哪个位置的物理量，通常用角度 (0° - 360°) 表示。
*   **相位差 (Phase Difference)**：两个频率相同的声波，如果起始时间不同，就会产生相位差。这是 **回声消除 (AEC)** 和 **主动降噪 (ANC)** 的核心物理基础。

---

## 3. 声学现象 (Acoustic Phenomena)

### 3.1 干涉 (Interference)
当两列或多列频率相同的声波在空间相遇时，会产生叠加效果：
*   **相长干涉 (Constructive Interference)**：两波相位相同，叠加后振幅增大。
*   **相消干涉 (Destructive Interference)**：两波相位相差 180°，叠加后振幅减小甚至抵消（主动降噪原理）。

```mermaid
graph TD
    subgraph "相消干涉 (Destructive Interference)"
        A1[原始声波 Phase: 0°]
        A2[反相声波 Phase: 180°]
        A3[叠加结果: 静音/衰减]
        A1 + A2 --> A3
    end
```

### 3.2 衍射与反射 (Diffraction & Reflection)
*   **反射**：声波遇到大于其波长的障碍物时会发生反射（回声、混响的基础）。
*   **衍射**：声波绕过障碍物继续传播的现象。频率越低，衍射能力越强。

---

## 4. 分贝系统 (The Decibel System)

由于人耳对声音强度的感知是非线性的（更接近对数关系），且声音能量跨度极大，物理学上使用 **分贝 (dB)** 这一对数单位。

### 4.1 为什么使用对数？
*   人耳能听到的最弱声音与能忍受的最强声音，能量比值高达 $1: 10^{12}$。使用对数可以将这个范围压缩到 0 - 120 dB。

### 4.2 声压级 (SPL, Sound Pressure Level)
衡量空气中声音强度的常用单位：
$$L_p = 20 \log_{10} \left( \frac{P}{P_0} \right)$$
*   $P$：测得的有效声压。
*   $P_0$：参考声压（通常为 $2 \times 10^{-5}$ Pa，即人耳能听到的最小声压）。

---

## 5. 关键参考 (References)

1.  *Principles of Vibration and Sound* - Thomas D. Rossing
2.  [Sound - Wikipedia](https://en.wikipedia.org/wiki/Sound)
3.  [The Speed of Sound - Engineering Toolbox](https://www.engineeringtoolbox.com/speed-sound-d_82.html)

---
*Next Topic: [心理声学 (Psychoacoustics)](./02-Psychoacoustics.md)*
```

- [ ] **Step 2: Commit the file**

Run:
```bash
git add 01-Acoustics-Digital-Audio/01-Sound-Wave-Physics.md
git commit -m "feat: add first acoustics chapter - Sound Wave Physics"
```

---
End of plan.
