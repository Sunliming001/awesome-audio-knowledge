# SoundWire 总线

SoundWire 是 MIPI 联盟于 2014 年发布的新一代音频接口标准，旨在取代 I2S/PDM/SLIMbus，实现单总线上的音频数据传输 + 设备控制 + 动态电源管理。高通 WCD93xx 系列 Codec 和 WSA88xx SmartPA 已全面采用 SoundWire。

---

## 1. SoundWire vs 传统接口

```
SoundWire 解决了传统接口的痛点:

  ┌────────────────┬──────────┬──────────┬──────────┬──────────────┐
  │ 维度           │ I2S      │ SLIMbus  │ PDM      │ SoundWire    │
  ├────────────────┼──────────┼──────────┼──────────┼──────────────┤
  │ 信号线数       │ 3-4      │ 2        │ 2        │ 2            │
  │ 数据+控制      │ 仅数据   │ 数据+控制│ 仅数据   │ 数据+控制    │
  │ 多设备挂载     │ ❌       │ ✅       │ ❌       │ ✅           │
  │ 动态带宽分配   │ ❌       │ ✅       │ ❌       │ ✅           │
  │ 电源管理       │ 无       │ 基本     │ 无       │ 完善 (Clock  │
  │                │          │          │          │ Stop/Pause)  │
  │ 最大带宽       │ ~50Mbps  │ ~28Mbps  │ ~5Mbps   │ ~100Mbps     │
  │ 标准化组织     │ Philips  │ MIPI     │ 无       │ MIPI         │
  │ 目前状态       │ 广泛使用 │ 被取代中 │ DMIC 用  │ 高通主推     │
  └────────────────┴──────────┴──────────┴──────────┴──────────────┘
```

---

## 2. SoundWire 物理层

```
SoundWire 仅使用 2 根信号线:

  ┌───────────────────────────────────────────────────┐
  │ 信号线           说明                             │
  ├───────────────────────────────────────────────────┤
  │ SWR_CLK (Clock)  主时钟, Master 提供              │
  │                  频率: 0.5 ~ 12.288 MHz           │
  │ SWR_DATA (Data)  双向数据线 (DDR)                 │
  │                  在 CLK 上升沿和下降沿都传数据     │
  └───────────────────────────────────────────────────┘
  
  拓扑结构:
    一条 SoundWire Bus 上可挂载:
      1 个 Master (SoC / Codec)
      最多 11 个 Slave 设备
      
    高通手机典型拓扑:
      SoC LPAIF ── SWR Master 0 ── WCD9385 (Codec, Slave)
                 ── SWR Master 1 ── WSA8845 #1 (SmartPA, Slave)
                                  ── WSA8845 #2 (SmartPA, Slave)
                                  
  帧结构 (Frame):
    每帧 = Control Word + Data Slots
    Control Word: 用于寄存器读写、设备枚举、中断
    Data Slots: 用于 PCM 音频数据传输
    → 控制和数据共用同一总线, 无需额外 I2C/SPI
```

---

## 3. SoundWire 协议层

### 3.1 设备枚举

```
SoundWire 设备自动枚举流程:

  1. Master 发送 PING 命令
     → 检测总线上有哪些 Slave 响应
     
  2. Slave 响应自己的 Device ID
     → Device ID = 48-bit 唯一标识
     → 包含: Manufacturer ID + Part ID + Version
     
  3. Master 分配 Device Number (1-11)
     → 后续通信用 Device Number 寻址
     
  4. Master 读取 Slave 的 Data Port 能力
     → 支持多少 Data Port
     → 每个 Port 的采样率/位宽/声道数
     
  5. Master 配置 Data Port
     → 分配 Stream (Playback/Capture)
     → 配置 Transport 参数 (Block Offset, Sample Interval)
     
  vs I2C 设备地址: 
    I2C = 7-bit 固定地址, 容易冲突
    SoundWire = 48-bit 唯一 ID, 自动枚举, 不冲突
```

### 3.2 寄存器访问

```
SoundWire 内嵌寄存器读写 (替代 I2C):

  写寄存器:
    Master → Control Word: [DevNum][Write][RegAddr][Data]
    → 在音频传输间隙插入, 不影响音频数据
    
  读寄存器:
    Master → Control Word: [DevNum][Read][RegAddr]
    Slave  → 下一帧返回: [Data]
    
  优势:
    - 无需额外 I2C 总线
    - 寄存器访问与音频传输共享同一物理层
    - 支持批量读写 (Bulk Transfer)
    
  Linux 中的实现:
    SoundWire Slave 注册为 regmap 设备
    → Codec 驱动直接用 snd_soc_component_read/write()
    → 底层通过 SoundWire Control Word 传输
```

---

## 4. SoundWire 电源管理

```
SoundWire Clock Stop 机制:

  ┌────────────────────────────────────────────────────┐
  │ 状态            说明                  功耗         │
  ├────────────────────────────────────────────────────┤
  │ Active          正常传输              全速         │
  │ Clock Stop 0    时钟暂停, Slave 保持  极低 (~µW)   │
  │   (De0)         寄存器状态, 快速恢复  恢复 <100µs  │
  │ Clock Stop 1    时钟停止, Slave 可    最低         │
  │   (De1)         进入深度睡眠          恢复 ~1ms    │
  └────────────────────────────────────────────────────┘
  
  使用场景:
    播放音乐时 → Active
    暂停播放 → Clock Stop 0 (快速恢复, 用户感知不到)
    息屏待机 → Clock Stop 1 (最省电)
    语音唤醒 → Codec 保持低功耗监听, SWR 进入 Clock Stop
    
  DAPM 集成:
    ASoC DAPM Widget 状态变化时自动触发 Clock Stop/Resume
    → 用户无感知的动态功耗管理
```

---

## 5. Linux SoundWire 驱动架构

```
Linux SoundWire 驱动栈:

  用户空间
    ├── ALSA 应用 / AudioFlinger
    └── libasound / TinyALSA
  ────────────────────────────────────────
  内核空间
    ├── ALSA Core / ASoC Framework
    ├── Codec 驱动 (如 wcd938x.c)
    │     → 使用 regmap-soundwire 访问寄存器
    │     → 注册 DAPM Widget / Route
    ├── SoundWire Bus Driver (drivers/soundwire/)
    │     ├── bus.c          → 总线注册/设备枚举
    │     ├── slave.c        → Slave 设备管理
    │     ├── stream.c       → Stream 配置
    │     ├── master.c       → Master 控制器接口
    │     └── sysfs.c        → debugfs 接口
    └── SoundWire Master Controller Driver
          → 高通: qcom-soundwire.c
          → Intel: intel-soundwire.c
  ────────────────────────────────────────
  硬件
    SoC SWR Master ──── SWR Bus ──── WCD9385 / WSA8845
    
  关键源码:
    drivers/soundwire/           ← SoundWire 核心框架
    techpack/audio/asoc/codecs/  ← 高通 Codec SWR 驱动
```

### 5.1 调试命令

```bash
# 查看 SoundWire 设备
adb shell ls /sys/bus/soundwire/devices/
# sdw:0:0217:0110:00:0   → Master 0, Slave WCD9385
# sdw:1:0217:0204:00:0   → Master 1, Slave WSA8845

# 查看 SoundWire Master 状态
adb shell cat /sys/kernel/debug/soundwire/master-0/status
# State: active / clock_stop

# 查看 Slave 设备信息
adb shell cat /sys/bus/soundwire/devices/sdw:0:0217:0110:00:0/device_id
# Manufacturer: 0x0217 (Qualcomm)
# Part: 0x0110 (WCD9385)

# SoundWire 传输错误统计
adb shell cat /sys/kernel/debug/soundwire/master-0/statistics
# Bus clash count, Parity errors, etc.

# 常见问题:
# 1. Slave 未枚举 → SWR_CLK/SWR_DATA 信号线问题
# 2. 寄存器读写失败 → 总线忙或 Slave 未响应
# 3. Clock Stop 恢复慢 → 检查 Slave 唤醒时间配置
```

---

## 6. 关键参考

1. [MIPI SoundWire Specification v1.2](https://www.mipi.org/specifications/soundwire)
2. Linux Kernel: `drivers/soundwire/` — SoundWire core framework
3. [Intel SoundWire Documentation](https://www.kernel.org/doc/html/latest/driver-api/soundwire/)
4. Qualcomm SoundWire Master/Slave Driver — `techpack/audio/`

---
*Back: [接口与总线导航](./README.md)*
