# GEF：通用音效框架 (Generic Effects Framework)

GEF (Generic Effects Framework) 是高通 AudioReach 体系中连接 HLOS 应用层与 ADSP 音效模块的桥梁。它允许上层应用（如音效 App）通过标准化接口下发参数到 DSP 侧的音效处理模块，无需修改 Audio HAL。

---

## 1. GEF 在音频栈中的位置

```mermaid
graph TD
    subgraph HLOS ["HLOS (Android/Linux)"]
        APP["音效 App<br/>(如 Dolby / DTS)"]
        GEF_LIB["GEF Client Library<br/>(libgef.so)"]
        AGMS["AGMS / PAL"]
    end
    
    subgraph ADSP ["ADSP"]
        SPF["SPF Framework"]
        COPP["COPP 音效链<br/>(EQ → DRC → Spatial)"]
        POPP["POPP 音效链"]
    end
    
    APP -->|"GEF API"| GEF_LIB
    GEF_LIB -->|"参数打包"| AGMS
    AGMS -->|"GPR"| SPF
    SPF --> COPP
    SPF --> POPP
```

---

## 2. 核心概念

### 2.1 GEF 设计目标

*   **统一入口**：所有第三方音效厂商通过同一套 API 下发参数
*   **无需 HAL 修改**：参数直接穿透到 ADSP 模块，不经过 AudioFlinger
*   **Use Case 感知**：可根据不同音频用例 (Playback / VoIP / Record) 绑定不同音效配置
*   **持久化**：支持参数的保存与恢复

### 2.2 关键术语

| 术语 | 含义 |
|:---|:---|
| **GEF Config** | 一组音效参数的集合，绑定到特定 Use Case |
| **Effect ID** | 音效模块的唯一标识 (即 CAPI Module ID) |
| **Param ID** | 模块内部的参数标识 |
| **Subgraph Type** | 参数目标：COPP (设备侧) / POPP (流侧) |
| **Calibration Data** | ACDB 中存储的默认参数值 |

---

## 3. GEF 工作流程

### 3.1 参数下发路径

```mermaid
sequenceDiagram
    participant App as 音效 App
    participant GEF as GEF Library
    participant PAL as PAL/AGMS
    participant SPF as SPF (ADSP)
    participant MOD as CAPI Module
    
    App->>GEF: gef_set_param(effect_id, param_id, payload)
    GEF->>GEF: 参数序列化 + 路由决策
    GEF->>PAL: 通过 session handle 发送
    PAL->>SPF: GPR_CMD_SET_CFG
    SPF->>MOD: set_param(param_id, payload)
    MOD-->>SPF: ACK
    SPF-->>PAL: Response
    PAL-->>GEF: Success
    GEF-->>App: CAPI_EOK
```

### 3.2 参数获取路径

```mermaid
sequenceDiagram
    participant App as 音效 App
    participant GEF as GEF Library
    participant SPF as SPF (ADSP)
    participant MOD as CAPI Module
    
    App->>GEF: gef_get_param(effect_id, param_id)
    GEF->>SPF: GPR_CMD_GET_CFG
    SPF->>MOD: get_param(param_id)
    MOD-->>SPF: payload 返回
    SPF-->>GEF: Response + data
    GEF-->>App: 参数数据
```

---

## 4. GEF API 接口

### 4.1 初始化与销毁

```c
#include "gef_api.h"

/* 初始化 GEF 客户端 */
gef_handle_t gef_handle;
gef_status_t status = gef_init(&gef_handle);

/* 销毁 */
gef_deinit(gef_handle);
```

### 4.2 参数设置

```c
/* 设置音效参数 */
gef_param_info_t param_info = {
    .module_id   = 0x10001234,   /* 目标 CAPI 模块 ID */
    .param_id    = 0x10001235,   /* 参数 ID */
    .instance_id = 0,            /* 模块实例 (若拓扑中有多个相同模块) */
};

my_eq_config_t eq_cfg = {
    .band_count = 5,
    .bands = { /* ... */ },
};

status = gef_set_param(gef_handle, 
                       &param_info, 
                       (uint8_t *)&eq_cfg, 
                       sizeof(eq_cfg));
```

### 4.3 绑定到特定 Use Case

```c
/* 将配置绑定到播放场景 */
gef_usecase_info_t usecase = {
    .stream_type = GEF_STREAM_PLAYBACK,
    .device_id   = GEF_DEVICE_SPEAKER,
};

status = gef_set_param_for_usecase(gef_handle, 
                                    &usecase,
                                    &param_info, 
                                    (uint8_t *)&eq_cfg, 
                                    sizeof(eq_cfg));
```

---

## 5. GEF 与 ACDB 的关系

```mermaid
graph LR
    subgraph Design_Time ["设计时 (QACT)"]
        QACT["QACT 调参工具"]
        ACDB["ACDB 数据库<br/>(默认参数)"]
    end
    
    subgraph Runtime ["运行时"]
        GEF_RT["GEF Runtime"]
        PERSIST["持久化存储<br/>(覆写参数)"]
    end
    
    QACT --> ACDB
    ACDB -->|"开机加载默认值"| GEF_RT
    GEF_RT -->|"App 动态修改"| PERSIST
    PERSIST -->|"下次开机恢复"| GEF_RT
```

*   **ACDB 提供默认值**：出厂调音参数存储在 ACDB 中
*   **GEF 运行时覆写**：用户通过音效 App 调整后，GEF 将差异参数持久化
*   **优先级**：GEF 覆写 > ACDB 默认

---

## 6. 典型应用场景

### 6.1 第三方音效集成 (如 Dolby Atmos)

```
Dolby App → GEF API → ADSP Dolby Module (CAPI)
                            ↓
                    实时处理 Atmos 渲染
```

### 6.2 OEM 自定义 EQ

```
设置 App (Sound Settings) → GEF API → ADSP PEQ Module
    用户拖动 EQ 滑条              →  实时更新 Band Gain
```

### 6.3 车载多场景切换

```
驾驶模式 → GEF usecase = {PLAYBACK, DRIVER_SPEAKER}  → 驾驶位优化参数
影院模式 → GEF usecase = {PLAYBACK, ALL_SPEAKERS}     → 全车环绕参数
```

---

## 7. 调试方法

### 7.1 验证参数是否下达

```bash
# QXDM 过滤 GEF 相关日志
Filter: MSG_SSID_QDSP6 + "gef" 或 "set_param"

# 预期日志
"GEF: set_param module=0x10001234 param=0x10001235 size=20 SUCCESS"
```

### 7.2 常见问题

| 问题 | 原因 | 解决方案 |
|:---|:---|:---|
| 参数不生效 | Module 未在当前 graph 中加载 | 确认 Use Case 已启动对应拓扑 |
| 返回 MODULE_NOT_FOUND | Module ID 错误或未注册 AMDB | 核对 Module ID 与 AMDB |
| 持久化失败 | 文件系统权限问题 | 检查 `/data/vendor/audio/` 权限 |
| 参数被覆盖 | ACDB 重新加载覆写了 GEF 设置 | 调整加载顺序或使用 GEF persist |

---

## 8. GEF 与 Android AudioEffect 的关系

```
Android AudioEffect vs GEF 的定位:

  ┌─────────────────────────────────────────────────────────────┐
  │ Android AudioEffect Framework (Java/Native)                │
  │   → 标准 API: Equalizer, BassBoost, Virtualizer, ...      │
  │   → 工作在 AudioFlinger 层 (ARM 侧)                       │
  │   → 每个 AudioSession 可挂载不同 Effect                    │
  │   → 适用: App 级音效控制                                   │
  └─────────────────────────────────────────────────────────────┘
           │ 只能控制 ARM 侧 Effect
           │ 无法触及 ADSP 内部模块参数
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ GEF (Generic Effects Framework)                            │
  │   → 直接操作 ADSP SPF Graph 中的模块                       │
  │   → 绕过 AudioFlinger                                     │
  │   → 适用: 系统级音效、硬件级调参、OEM 定制                 │
  │   → 可做到 AudioEffect 做不到的事:                        │
  │       - 控制 ADSP 内部 EQ/DRC 参数                        │
  │       - 修改 Speaker Protection 阈值                       │
  │       - 切换不同 use case 的算法参数                       │
  │       - 持久化参数 (设备重启后保持)                        │
  └─────────────────────────────────────────────────────────────┘
  
  协作方式:
    App → AudioEffect API → Equalizer Effect → AudioFlinger 层处理
    OEM → GEF API → 直接下发参数到 ADSP Module → 硬件级处理
    
    两者可以共存: App 控制的 EQ 和 GEF 控制的 ADSP EQ 串联
```

---

## 9. GEF 参数调试全流程

### 9.1 端到端调试步骤

```bash
# Step 1: 确认 ADSP Graph 已启动
adb shell cat /proc/asound/card0/agm_dump
# 查看当前活跃的 Use Case 和 Graph

# Step 2: 查看 GEF 可配置的 Module 列表
adb shell dumpsys vendor.audio.effects | grep -i gef
# 或者通过 QXDM 过滤:
#   MSG_SSID_QDSP6 + "gef" → 看到 Module 注册列表

# Step 3: 使用 GEF API 设置参数 (通过 test app)
# GEF test tool (OEM 提供):
gef_test set_param \
    --module_id 0x10001234 \
    --param_id 0x10001235 \
    --payload "01 00 00 00 ..."  # 二进制参数

# Step 4: 验证参数已到达 ADSP
# QXDM 日志:
#   "GEF: set_param module=0x10001234 param=0x10001235 size=20 SUCCESS"
# 或:
adb shell cat /sys/kernel/debug/aud_dev/gef_status

# Step 5: 验证音频效果
# 用 Audio Precision 或 REW 测量频响变化
# 播放正弦扫频 → 录音 → 对比 EQ 前后频响

# Step 6: 持久化参数
gef_test persist \
    --module_id 0x10001234 \
    --usecase "PLAYBACK_SPEAKER"
# 参数保存到 /data/vendor/audio/gef_persist/
# 下次启动自动加载
```

### 9.2 GEF 参数格式

```
GEF 参数的二进制格式:

  ┌────────────────────────────────────────┐
  │ Param Header (8 bytes)                │
  │   Module Instance ID : uint32_t       │
  │   Param ID           : uint32_t       │
  ├────────────────────────────────────────┤
  │ Param Payload (variable length)       │
  │   具体格式由 Module 定义              │
  │                                        │
  │   例: 5-band PEQ 参数                 │
  │     num_bands      : uint32_t = 5     │
  │     band[0].type   : uint32_t (Peak)  │
  │     band[0].freq   : uint32_t (100Hz) │
  │     band[0].gain   : int32_t (-3dB)   │
  │     band[0].Q      : uint32_t (Q=1.0) │
  │     band[1]...                         │
  └────────────────────────────────────────┘
  
  注意:
    - 所有值使用 Little-Endian
    - 增益通常用 Q27 或 Q28 定点数表示
    - 频率单位: Hz (uint32_t)
    - Q 值: Q8 定点数 (如 Q=1.0 → 0x100)
```

---

## 10. GEF 与 ACDB/QACT 的配合

```
GEF / ACDB / QACT 三者关系:

  QACT (调参工具, PC 端)
    │
    │ 导出
    ▼
  ACDB (Audio Calibration Database)
    │ → 存储默认参数 (出厂配置)
    │ → 每个 <Device, Topology, UseCase> 组合一套参数
    │ → 文件: /vendor/etc/acdbdata/*.acdb
    │
    │ 启动时加载
    ▼
  ADSP SPF Module ← 默认参数
    │
    │ 运行时覆盖
    ▼
  GEF Set Param
    │ → 在 ACDB 默认值之上动态修改
    │ → 优先级: GEF > ACDB
    │ → 可选: 持久化到文件系统
    
  典型工作流:
    1. 硬件调试阶段: QACT 反复调参 → 导出 ACDB
    2. 量产阶段: ACDB 固化到 /vendor 分区
    3. 运行时定制: GEF 动态修改特定参数 (如用户自定义 EQ)
    4. OTA 升级: 更新 ACDB 文件, GEF persist 参数保留
```

---

## 11. 关键参考 (References)

1.  80-VN500-24: *Generic Effects Framework (GEF) for AudioReach*
2.  80-VN500-11: *AudioReach LA Customizations*
3.  80-VN500-13: *AudioReach Use Case Customizations*
4.  80-VN500-4: *AudioReach SPF Modules API Reference*
5.  80-VN500-1: *AudioReach Architecture Overview*
