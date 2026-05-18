# AudioPolicy 策略管理深度解析

`AudioPolicy` 是 Android 音频系统的“大脑”，负责制定路由、音量和焦点的决策。

---

## 1. 启动与初始化全链路 (Initialization)

`AudioPolicy` 服务紧随 `AudioFlinger` 启动。

### 1.1 启动入口：onFirstRef()
在 `AudioPolicyService` 被实例化时，会触发其 `onFirstRef()` 生命周期：
1.  **创建命令线程**：启动 `AudioCommandThread` (ApmAudio/ApmOutput)，负责执行耗时的策略更新。
2.  **加载 Manager**：执行 `loadAudioPolicyManager()`。
3.  **实例化 APM**：通过厂商动态库创建 `mAudioPolicyManager`。

### 1.2 配置文件加载逻辑
`AudioPolicyManager` 在构造函数中执行 `loadConfig()`：
1.  **寻找 XML**：按顺序扫描 `/odm/etc`, `/vendor/etc`, `/system/etc` 下的 `audio_policy_configuration.xml`。
2.  **反序列化**：将 XML 标签转换为 C++ 对象（`HwModule`, `IOProfile`, `DeviceDescriptor`）。

---

## 2. 路由决策逻辑 (Strategy -> Device)

系统如何决定音频流发往扬声器还是耳机？这是一套基于 **Usage -> Strategy -> Device** 的推导过程。

### 2.1 核心计算函数：getDeviceForStrategy
```cpp
// AudioPolicyManager.cpp 逻辑
audio_devices_t AudioPolicyManager::getDeviceForStrategy(routing_strategy strategy) {
    // 优先级排序示例：
    // 通话时：蓝牙耳机(SCO) > 有线耳机 > 听筒
    if (strategy == STRATEGY_PHONE) {
        if (mBluetoothAvailable) return AUDIO_DEVICE_OUT_BLUETOOTH_SCO;
        if (mWiredHeadsetAvailable) return AUDIO_DEVICE_OUT_WIRED_HEADSET;
        return AUDIO_DEVICE_OUT_EARPIECE;
    }
}
```

---

## 3. 音量控制体系

Android 使用非线性的**对数映射**管理音量。

### 3.1 配置文件加载
系统通过 `EngineBase::loadAudioPolicyEngineConfig()` 加载 `audio_policy_volumes.xml`。
*   **Volume Group**：将多个 Context 归类。
*   **Index to dB**：定义了用户界面数值（如 0-15）到物理增益（dB）的转换曲线。

---

## 4. 专家调试：配置与 Dump

`adb shell dumpsys media.audio_policy`
*   **mAvailableOutputDevices**：查看系统是否识别到了硬件（如 USB 声卡、蓝牙耳机）。
*   **mOutputs**：查看当前每一个活跃流的路由路径、采样率和 Flags。

---
*下一模块：[07. Audio HAL 接口规范](./07-AudioHAL.md)*
