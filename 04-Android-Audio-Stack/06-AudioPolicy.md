# AudioPolicy 策略管理深度解析

`AudioPolicy` 是 Android 音频系统的“大脑”，负责回答一个终极问题：**“在这个时刻，这路声音，应该发往哪个物理设备？”**

---

## 1. 启动与初始化全链路 (Initialization)

`AudioPolicy` 服务紧随 `AudioFlinger` 之后由 `audioserver` 进程启动。

### 1.1 生命周期入口：onFirstRef()
在 `AudioPolicyService` 实例化时，会执行关键配置：
1.  **创建命令线程**：启动 `AudioCommandThread`，负责执行异步的路由切换。
2.  **加载 Manager**：调用 `loadAudioPolicyManager()`。
3.  **实例化内核**：系统根据 `libaudiopolicyengine.so` 的厂商实现，动态创建 `AudioPolicyManager` 实例。

### 1.2 配置文件解析 (Serialization)
`AudioPolicyManager` 在构造函数中调用 `loadConfig()`。其核心是 `PolicySerializer`，它递归地将 XML 标签转换为 C++ 对象模型。

| XML 标签 | C++ 实体类 | 核心职责 |
| :--- | :--- | :--- |
| `<module>` | `HwModule` | 代表一个硬件库（如 `primary`, `usb`）。 |
| `<mixPort>` | `IOProfile` | 代表流入口。定义了采样率、格式及 `maxActiveCount`。 |
| `<devicePort>`| `DeviceDescriptor` | 代表物理设备（如 `SPEAKER`, `WIRED_HEADSET`）。 |
| `<route>` | `AudioRoute` | 定义 `mixPort` 到 `devicePort` 的拓扑通路。 |

---

## 2. 路由决策逻辑深度拆解

这是一套基于 **Usage -> Strategy -> Device** 的推导模型。

### 2.1 推导三部曲
1.  **Match Strategy**：App 定义 `Usage` (如 `USAGE_MEDIA`)，系统查询 `getStrategyForUsage()` 得到策略（如 `STRATEGY_MEDIA`）。
2.  **Match Device**：调用核心算法 `getDeviceForStrategy(strategy)`。
    *   **优先级逻辑**：例如在 `STRATEGY_PHONE` 下，优先级顺序为：蓝牙 SCO > 有线耳机 > 听筒 > 扬声器。
3.  **Choose Output**：根据选择的 Device，在 `HwModule` 集合中寻找最匹配的 `IOProfile`，最终找到对应的 `PlaybackThread`。

---

## 3. 音量控制体系 (Volume Systems)

Android 使用**非线性对数映射**来匹配人耳感官。

### 3.1 配置文件加载
系统通过 `EngineBase::loadAudioPolicyEngineConfig()` 加载 `audio_policy_volumes.xml`。
*   **Volume Group**：将多个相似 Context 归类统一调节。
*   **Index to dB**：定义了用户界面数值（如 0-15 级）到物理增益（dB）的转换表。
*   **增益计算**：$Amplification = 10^{(\text{dB}/20)}$。

---

## 4. 关键交互：APM 与 AudioFlinger

当路由发生变化时（如：插拔耳机），APM 通过 `AudioPolicyClientInterface` 发起联动：
1.  **`loadHwModule()`**：通知 Flinger 加载新的厂商库。
2.  **`openOutput()`**：指示 Flinger 创建对应的 `PlaybackThread`。
3.  **`setParameters()`**：直接向 HAL 发送键值对（如 `routing=2`），触发硬件开关切换。

---

## 5. 专家调试与配置

### 5.1 配置文件路径搜索
系统按以下优先级查找 `audio_policy_configuration.xml`：
1. `/odm/etc/`
2. `/vendor/etc/audio/sku_<variant>/`
3. `/vendor/etc/`
4. `/system/etc/`

### 5.2 调试神级命令
`adb shell dumpsys media.audio_policy`
*   **mAvailableOutputDevices**：查看系统当前在线的硬件节点。
*   **mOutputs**：查看每一路活跃流的句柄、策略和最终出口。

---
*Next Topic: [Audio HAL 接口规范](./07-AudioHAL.md)*
