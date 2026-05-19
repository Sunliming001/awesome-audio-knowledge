# PDM 脉冲密度调制 (Pulse Density Modulation)

PDM 是数字 MEMS 麦克风与 SoC 之间的标准接口。与 I2S 传输多位 PCM 数据不同，PDM 仅用 1 bit 表示音频——靠"1"的密度编码振幅。

---

## 1. PDM 调制原理

```
PDM 信号特征:

  采样率: 极高 (1.024 MHz ~ 4.8 MHz, 典型 3.072 MHz)
  位宽:   1 bit (只有 0 和 1)
  编码:   1 的密度 ∝ 信号振幅
  
  波形示意:
    模拟信号:   ╱‾‾‾‾‾╲            ╱‾╲
               ╱       ╲          ╱   ╲
    ──────────╱         ╲────────╱     ╲───────
    
    PDM 码流: 1110111101111110 0001000010000001 0110111011101110
              ← 高密度(正峰值) → ← 低密度(负峰值) → ← 中等密度 →
              
  vs PCM:
    PCM: 每个采样 = 16/24 bit 的精确数值
    PDM: 每个采样 = 1 bit, 但采样率高 64-256 倍
    → 信息量等效: PDM @3.072MHz ≈ PCM @48kHz/16bit
    
  Sigma-Delta (ΣΔ) 调制器:
    MEMS 麦克风内部使用 ΣΔ ADC 将模拟信号转换为 PDM
    → 量化噪声被推到高频 (Noise Shaping)
    → 低频信号保持高 SNR
```

---

## 2. PDM 信号线

```
PDM 接口只需 2 根线:

  ┌──────────────────────────────────────────────┐
  │ 信号线       说明                            │
  ├──────────────────────────────────────────────┤
  │ PDM_CLK      时钟 (SoC 提供, 1~5 MHz)       │
  │ PDM_DATA     数据 (MEMS 麦克风输出)          │
  └──────────────────────────────────────────────┘
  
  一根 DATA 线可传输 2 个麦克风信号!
    → 麦克风 A: 在 CLK 上升沿输出数据
    → 麦克风 B: 在 CLK 下降沿输出数据
    → 通过 L/R Select 引脚配置 (接 GND 或 VDD)
    
  典型连接:
    SoC PDM_CLK ──→ DMIC0_CLK, DMIC1_CLK
    SoC PDM_DATA0 ←── DMIC0 (上升沿) + DMIC1 (下降沿)
    SoC PDM_DATA1 ←── DMIC2 (上升沿) + DMIC3 (下降沿)
    → 2 根 DATA 线 = 4 个麦克风
```

---

## 3. SoC 侧抽取处理 (Decimation)

```
PDM → PCM 转换链 (在 SoC 的数字音频前端完成):

  PDM 码流 (1-bit @ 3.072 MHz)
    │
    ▼
  ┌──────────────────────────────────┐
  │ CIC 滤波器 (Cascaded            │
  │ Integrator-Comb)                 │
  │   → 抽取 (Decimation) 降低采样率│
  │   → 3.072MHz ÷ 64 = 48kHz       │
  │   → 输出: 多位 PCM              │
  │   → 但频响有 droop (衰减)       │
  └──────────────────────────────────┘
    │
    ▼
  ┌──────────────────────────────────┐
  │ 补偿 FIR 滤波器                 │
  │   → 修正 CIC 的频响 droop       │
  │   → 恢复平坦频响               │
  └──────────────────────────────────┘
    │
    ▼
  ┌──────────────────────────────────┐
  │ HPF (高通滤波器)                 │
  │   → 滤除 DC Offset              │
  │   → 截止频率: ~20 Hz            │
  └──────────────────────────────────┘
    │
    ▼
  PCM 输出 (16/24-bit @ 48kHz)
  → 送入 ADSP / AudioFlinger 处理
  
  WCD938x Codec 中的 PDM 路径:
    DMIC → PDM Interface → Decimation Filter → TX DMA → SoundWire
```

---

## 4. DMIC 配置实战

### 4.1 Device Tree 配置 (高通平台)

```dts
// DMIC 在 Device Tree 中的定义:
&wcd938x_codec {
    // DMIC 时钟频率配置
    qcom,cdc-dmic-clk-drv-strength = <2>;  // 驱动强度
    
    // DMIC GPIO 配置
    qcom,cdc-dmic01-gpios = <&wcd_dmic01_gpios 0 0>;
    qcom,cdc-dmic23-gpios = <&wcd_dmic23_gpios 0 0>;
};

// DMIC GPIO pinctrl
wcd_dmic01_gpios: wcd-dmic01-gpios {
    compatible = "qcom,msm-cdc-pinctrl";
    pinctrl-names = "aud_active", "aud_sleep";
    pinctrl-0 = <&wcd_dmic01_clk_active &wcd_dmic01_data_active>;
    pinctrl-1 = <&wcd_dmic01_clk_sleep &wcd_dmic01_data_sleep>;
};
```

### 4.2 DMIC 调试

```bash
# 检查 DMIC 是否被识别
adb shell tinymix | grep -i dmic
# "DMIC0" → 0=Off, 1=DMIC0, 2=DMIC1 ...

# 测试录音
adb shell tinycap /data/local/tmp/test.wav -D 0 -d 6 -c 1 -r 48000 -b 16
# -d 6: PCM device ID (DMIC)

# 常见问题:
# 1. 录音无声 → 检查 PDM_CLK 是否输出 (示波器)
# 2. 录音噪声大 → 检查 DMIC 供电 (1.8V), 检查 PCB 走线
# 3. 左右声道反 → L/R Select 引脚接错
# 4. 底噪偏高 → 增大 CIC 滤波器阶数, 或调整抽取比
```

---

## 5. PDM vs I2S 数字麦克风对比

| 维度 | PDM (DMIC) | I2S 数字麦克风 |
|:---|:---|:---|
| 信号线数 | 2 (CLK + DATA) | 3-4 (BCLK + LRCLK + SD) |
| 每线麦克风数 | 2 个 (上升/下降沿) | 1 个 |
| SoC 处理 | 需要 Decimation 滤波 | 直接 PCM, 无需滤波 |
| 功耗 | 较低 (~0.5mW) | 较高 (~1.5mW) |
| 布线复杂度 | 简单 (2 线) | 较复杂 |
| 音质上限 | 受 Decimation 滤波器影响 | 取决于内部 ADC |
| 主流应用 | 手机/笔电/IoT | 专业设备/车载 |

---

## 6. 关键参考

1. [PDM Microphones and Sigma-Delta ADC - Analog Devices](https://www.analog.com/en/technical-articles/pdm-microphones.html)
2. [Understanding PDM Digital Audio - TI](https://www.ti.com/lit/an/spraam7/spraam7.pdf)
3. Linux Kernel: `sound/soc/codecs/dmic.c` — Generic DMIC Codec driver

---
*Next: [SoundWire 总线](./04-SoundWire.md)*
