# 04. Android 音频架构 (Android Audio Stack)

本模块深度拆解 Android 系统中的音频全链路。

## 📖 章节导航

1.  **[Android 音频系统概览 (Overview)](./01-Overview.md)**
    *   全景架构、进程隔离模型与 AAOS 特殊集成。
2.  **[AudioService 系统管理中心](./02-AudioService.md)**
    *   Java 层管理总管，负责音量、焦点与设备连接状态。
3.  **[AudioTrack 播放流程解析](./03-AudioTrack.md)**
    *   Native 初始化调用栈与共享内存同步机制。
4.  **[AudioRecord 录音流程解析](./04-AudioRecord.md)**
    *   音频源选择与录音数据流向。
5.  **[AudioFlinger 混音引擎深度解析](./05-AudioFlinger.md)**
    *   线程模型、AudioMixer 算法与初始化流程。
6.  **[AudioPolicy 策略管理深度解析](./06-AudioPolicy.md)**
    *   策略决策算法、XML 配置加载与路由管理。
7.  **[Audio HAL 接口规范](./07-AudioHAL.md)**
    *   从 HIDL 到 AIDL 的演进与 ALSA 驱动对接。
8.  **[AudioEffect 音效框架深度解析](./08-AudioEffect.md)**
    *   音效链加载与 Buffer 传递同步。
9.  **[AudioFocus 音频焦点机制](./09-AudioFocus.md)**
    *   基于协作的多应用焦点竞争管理。

---
[返回主目录](../README.md)
