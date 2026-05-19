# TDM 时分复用 (Time Division Multiplexing)

当需要在一条物理 I2S 数据线上传输 2 个以上声道时（如车载 8ch、专业音频 16ch），TDM 是标准方案。TDM 本质上是 I2S DSP/PCM Mode 的多通道扩展。

---

## 1. TDM 基本原理

```
TDM 将每个采样周期 (1/Fs) 划分为多个时间槽 (Slot):

  一帧 = N 个 Slot, 每个 Slot 传输一个声道的数据
  
  Slot 0    Slot 1    Slot 2    Slot 3    Slot 4 ...
  ┌────────┬────────┬────────┬────────┬────────┐
  │  Ch 0  │  Ch 1  │  Ch 2  │  Ch 3  │  Ch 4  │ ...
  └────────┴────────┴────────┴────────┴────────┘
  ←──────────── 1 Frame (1/Fs) ────────────────→
  
  帧同步信号 (FSYNC):
    长帧模式 (Long Frame): FSYNC = Slot 0 持续时间 (类似 LRCLK)
    短帧模式 (Short Frame): FSYNC = 1 BCLK 宽度的脉冲 → 更常用
```

---

## 2. 关键参数计算

```
TDM Bitclock 计算:

  BCLK = Sample_Rate × Num_Slots × Slot_Width
  
  示例:
  ┌──────────────┬────────┬────────────┬───────────────┐
  │ 场景         │ Fs     │ Slots×Width│ BCLK          │
  ├──────────────┼────────┼────────────┼───────────────┤
  │ Stereo I2S   │ 48kHz  │ 2 × 32bit │ 3.072 MHz     │
  │ 车载 4ch     │ 48kHz  │ 4 × 32bit │ 6.144 MHz     │
  │ 车载 8ch     │ 48kHz  │ 8 × 32bit │ 12.288 MHz    │
  │ 车载 16ch    │ 48kHz  │ 16 × 32bit│ 24.576 MHz    │
  │ 高保真 8ch   │ 96kHz  │ 8 × 32bit │ 24.576 MHz    │
  │ 专业音频 32ch│ 48kHz  │ 32 × 32bit│ 49.152 MHz    │
  └──────────────┴────────┴────────────┴───────────────┘
  
  Slot Width vs Sample Width:
    Slot Width = 32-bit (固定槽宽, 含 padding)
    Sample Width = 16/24-bit (实际有效数据)
    → 16-bit 数据放在 32-bit Slot 中, 高位对齐, 低位补 0
```

---

## 3. TDM Slot Mapping (通道映射)

```
Slot Mapping = 哪个声道放在哪个 Slot 位置:

  高通平台 TDM 配置示例:
    TDM RX (播放):
      Slot 0 → Front Left
      Slot 1 → Front Right  
      Slot 2 → Center
      Slot 3 → LFE (低音炮)
      Slot 4 → Rear Left
      Slot 5 → Rear Right
      Slot 6 → Side Left
      Slot 7 → Side Right
      
    TDM TX (录音):
      Slot 0 → Mic 1
      Slot 1 → Mic 2
      Slot 2 → Reference (AEC 参考)
      Slot 3 → IV-Sense Left
      
  Slot Mask 表示法 (bitmask):
    使用哪些 Slot: slot_mask = 0xFF (8 个 Slot 全用)
    只用 Slot 0,1: slot_mask = 0x03
    只用 Slot 0,2: slot_mask = 0x05
```

### 3.1 高通平台 TDM 配置

```c
// 高通 Machine Driver 中的 TDM 配置:
// techpack/audio/asoc/msm-dai-q6-v2.c

// 设置 TDM 参数
static int msm_tdm_snd_hw_params(struct snd_pcm_substream *substream,
                                  struct snd_pcm_hw_params *params) {
    struct snd_soc_pcm_runtime *rtd = substream->private_data;
    struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
    
    int slot_width = 32;
    int num_slots = 8;
    unsigned int slot_mask = 0xFF;  // 使用全部 8 个 slot
    unsigned int slot_offset[8] = {0, 4, 8, 12, 16, 20, 24, 28}; // 字节偏移
    
    ret = snd_soc_dai_set_tdm_slot(cpu_dai, 
            slot_mask,      // TX slot mask
            slot_mask,      // RX slot mask
            num_slots,      // 总 slot 数
            slot_width);    // 每 slot 位宽
    
    return ret;
}
```

### 3.2 Device Tree TDM 配置

```dts
// 高通平台 DTS 中的 TDM 端口定义:
&q6core {
    pri_tdm: primary-tdm {
        compatible = "qcom,msm-dai-tdm";
        qcom,msm-cpudai-tdm-group-id = <0x3700>;  // Primary TDM RX
        qcom,msm-cpudai-tdm-group-num-ports = <1>;
        qcom,msm-cpudai-tdm-clk-rate = <12288000>; // BCLK = 12.288MHz
        qcom,msm-cpudai-tdm-sync-mode = <0>;       // Short Sync
        qcom,msm-cpudai-tdm-sync-src = <1>;         // Internal
        qcom,msm-cpudai-tdm-data-out = <0>;         // Data Out
        qcom,msm-cpudai-tdm-invert-sync = <0>;
        qcom,msm-cpudai-tdm-data-delay = <1>;       // 1 BCLK delay
        
        dai_pri_tdm_rx_0: qcom,msm-dai-q6-tdm-pri-rx-0 {
            compatible = "qcom,msm-dai-q6-tdm";
            qcom,msm-cpudai-tdm-dev-id = <0x3700>;
            qcom,msm-cpudai-tdm-data-align = <0>; // MSB aligned
        };
    };
};
```

---

## 4. TDM 与 I2S 的关系

```
I2S 是 TDM 的特殊情况 (2 Slot TDM):

  I2S Standard = TDM Long Frame, 2 slots, 1 BCLK delay
  I2S Left-J   = TDM Long Frame, 2 slots, 0 delay
  DSP/PCM Mode = TDM Short Frame, N slots
  
  在 Linux ASoC 中:
    2ch I2S  → snd_soc_dai_set_fmt(SND_SOC_DAIFMT_I2S)
    多ch TDM → snd_soc_dai_set_fmt(SND_SOC_DAIFMT_DSP_B)
              + snd_soc_dai_set_tdm_slot()
```

---

## 5. 常见 TDM 问题

| # | 问题 | 原因 | 解决 |
|:---|:---|:---|:---|
| 1 | 只有部分声道有声 | Slot mask 配置不全 | 检查 slot_mask 是否覆盖所有声道 |
| 2 | 声道顺序错乱 | Slot offset 配错 | 核对 slot_offset 数组 |
| 3 | 声音变调/异常 | BCLK 频率不对 | 确认 BCLK = Fs × Slots × SlotWidth |
| 4 | 杂音/底噪 | 数据对齐问题 | 检查 data-delay (0 或 1 BCLK) |
| 5 | 多设备不同步 | 时钟域不一致 | 确保所有设备使用同一 BCLK 源 |

```bash
# TDM 调试命令 (高通平台)
adb shell cat /proc/asound/card0/pcm*p/sub0/hw_params  # 查看 PCM 参数
adb shell tinymix | grep -i tdm    # 查看 TDM 相关 mixer controls
adb shell cat /proc/asound/card0/pcm*p/sub0/status     # 运行状态
```

---

## 6. 关键参考

1. [Understanding TDM - Texas Instruments Application Note](https://www.ti.com/lit/an/slaa701/slaa701.pdf)
2. Linux Kernel: `include/sound/soc-dai.h` — TDM slot API
3. Qualcomm AudioReach TDM Configuration Guide

---
*Next: [PDM 脉冲密度调制](./03-PDM.md)*
