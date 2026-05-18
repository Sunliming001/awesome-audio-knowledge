# 高通 ADSP 拓扑与调试 (ADSP Topology & Debugging)

ADSP (Audio Digital Signal Processing) 是高通 SoC 的音频大脑。本章深入探讨其内部通信协议、拓扑架构及专家级调试手段。

---

## 1. 通信协议深度解析：APR vs. GPR

AP (应用处理器) 与 DSP 之间的“通话”通过特定的数据包路由协议实现。

### 1.1 APR (Asynchronous Packet Router) - Elite 架构
*   **路由寻址**：基于 **Service ID** 和 **Port ID**。
*   **场景**：传统手机机型和旧版 Elite 驱动。

### 1.2 GPR (Graph Packet Router) - AudioReach 架构
*   **路由寻址**：基于 **Domain ID** 和 **Instance ID (IID)**。
*   **灵活性**：支持动态的图 (Graph) 拓扑寻址，更适合复杂的车载多音区场景。

```mermaid
graph LR
    subgraph "AP Layer"
        ALSA_Drv[ALSA Driver] --> IPC[Shared Memory / GLINK]
    end
    
    subgraph "ADSP Layer"
        IPC --> Router[GPR / APR Router]
        Router --> Service_Manager[Service Manager]
        Service_Manager --> Modules[EQ / Mixer / Vol]
    end
```

---

## 2. DSP 拓扑组件 (Topology Components)

理解拓扑结构是排查“无声”或“音质差”的关键。

*   **POPP (Per Object Processing Path)**：
    *   **位置**：靠近应用端（Decoder 之后）。
    *   **任务**：处理流特定的算法（如：不同音乐 App 的独立 EQ）。
*   **COPP (Common Object Processing Path)**：
    *   **位置**：混音之后，硬件输出之前。
    *   **任务**：处理设备特定的算法（如：扬声器保护、多声道下混）。
*   **AFE (Audio Front End)**：
    *   **位置**：最底层。
    *   **任务**：对接物理 I2S/TDM 接口，管理硬件采样率和时钟同步。

---

## 3. Multi-DSP 框架：负载均衡与低功耗

高通高性能 SoC（如 SA8295）通常集成了多个 Hexagon DSP，以应对海量的并发任务。

### 3.1 DSP 职责分工
1.  **LPASS aDSP (Main)**：处理核心音频流（Music, Voice Call）、3A 算法、混音。
2.  **mDSP (Modem DSP)**：处理 4G/5G 调制解调相关的音频编码。
3.  **sDSP (Sensor DSP)**：处理极低功耗的任务，如 **VAD (语音活动检测)** 和 **传感器融合**。

### 3.2 跨 DSP 通信
通过 **GLINK** 或 **Shared Memory** 实现。当 sDSP 检测到人声唤醒词时，会发送信号给 aDSP，由 aDSP 启动完整的 ASR 交互链路。

---

## 4. TDM 接口与 Slot 映射实战

在车载硬件中，SoC 与外部 Codec（如 Mercury）通常通过 TDM 8ch 或 16ch 连接。

### 4.1 Slot 定义示例
*   **Overall Format**：8 Slot / Channel, 32 BitsPerSlot, MSB Justified.
*   **Mapping (DINs)**：
    *   Slot 0-1：前排 MIC。
    *   Slot 2-3：后排 MIC。
    *   Slot 4-7：回声参考信号 (Echo Reference)。

### 4.2 常见 Mixer 配置
使用 `tinymix` 验证 TDM 状态：
```bash
tinymix 'QUAT_TDM_RX_0 Channels' 'Eight'
tinymix 'QUAT_TDM_RX_0 SlotWidth' '32'
```

---

## 5. 专家调试实战

### 3.1 关键调试节点 (PCM Dump)
高通 DSP 支持在路径的任何位置进行数据 Dump。
1.  **POPP Input**：确认 App 发来的数据是否正确。
2.  **COPP Output**：确认混音和音效处理后是否失真。
3.  **AFE RX/TX**：确认最终送往硬件的数据状态。

### 3.2 常用 adb 调试命令
```bash
# 查看 ADSP 是否崩溃 (SSR 状态)
adb shell cat /sys/kernel/debug/msm_subsys/adsp

# 查看音频驱动状态 (高通专有路径)
adb shell cat /sys/kernel/debug/audio_reach/graph_info

# 查看实时 DSP 负载 (需配合调试固件)
adb shell "echo 1 > /sys/kernel/debug/audio_reach/perf_monitor"
```

---

## 4. QACT 可视化连线调试

使用 QACT (Qualcomm Audio Configuration Tool) 时，开发者可以直接看到当前的 **Graph 拓扑**。
*   **实时微调**：点击 EQ 模块，拖动曲线，DSP 内部寄存器会通过 GPR 协议瞬间更新，实现“所调即所得”。

---

## 5. 关键参考 (References)

1.  [Qualcomm Hexagon DSP SDK Documentation](https://developer.qualcomm.com/software/hexagon-dsp-sdk)
2.  *High-Performance Audio on Qualcomm Mobile Platforms* - Industry Whitepaper
