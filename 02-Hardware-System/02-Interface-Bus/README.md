# 数字音频接口与总线 (Digital Audio Interfaces & Bus)

在音频 SoC 设计中，芯片间的数据交换（Inter-IC Communication）主要依赖以下几种串行总线。本模块拆分为四个独立章节深入讲解。

---

## 章节导航

1. **[I2S 总线详解 (Inter-IC Sound)](./01-I2S.md)**
   * 信号线定义（BCLK / LRCLK / SD / MCLK）与时钟关系。
   * 四种数据对齐格式（Standard / Left-J / Right-J / DSP Mode）。
   * 主从模式与 Linux ASoC `dai_fmt` 配置。
   * 常见 I2S 问题调试（无声、变调、声道反转）。

2. **[TDM 时分复用 (Time Division Multiplexing)](./02-TDM.md)**
   * Slot / Frame 结构与 BCLK 计算公式。
   * Slot Mapping 通道映射与 bitmask。
   * 高通平台 TDM 配置（Machine Driver + Device Tree）。
   * 常见 TDM 问题排查。

3. **[PDM 脉冲密度调制 (Pulse Density Modulation)](./03-PDM.md)**
   * Sigma-Delta 调制原理与 Noise Shaping。
   * 一线双麦（上升沿/下降沿复用）。
   * SoC 侧抽取处理链（CIC → 补偿 FIR → HPF）。
   * DMIC Device Tree 配置与调试命令。

4. **[SoundWire 总线](./04-SoundWire.md)**
   * SoundWire vs I2S/SLIMbus/PDM 全面对比。
   * 2 线物理层与双向 DDR 传输。
   * 设备自动枚举、寄存器访问（替代 I2C）。
   * Clock Stop 电源管理机制。
   * Linux SoundWire 驱动架构与调试。

---

## 接口选型速查

| 场景 | 推荐接口 | 说明 |
|:---|:---|:---|
| SoC ↔ Codec (2ch) | I2S | 最简单、最通用 |
| SoC ↔ 外部 DSP (多ch) | TDM | 单线多通道 |
| SoC ↔ DMIC | PDM | 2 线支持双麦 |
| SoC ↔ Codec + SmartPA | SoundWire | 数据+控制+电源管理一线搞定 |
| 车载 SoC ↔ 远端节点 | A2B | 长距离、菊花链 (见 [车载硬件](../04-Automotive-Hardware.md)) |

---
[返回硬件系统](../README.md) | [返回主目录](../../README.md)
