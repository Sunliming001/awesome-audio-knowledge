# 07. 高通平台专题 (Qualcomm Platform)

本模块深入剖析高通 (Qualcomm) 平台的音频子系统设计，这是目前移动端及车载市场占有率最高的音频方案。

## 📖 章节导航

1.  **[AudioReach 架构深度解析](./01-AudioReach-Deep-Dive.md)**
    *   AudioReach 模块化设计理念。
    *   核心对象：Graph, Subgraph, Container, Module。
    *   GPR (Graph Packet Router) 通信协议。

2.  **[高通 ADSP 拓扑与调试](./02-ADSP-Topology.md)**
    *   ADSP 内部通信路径与 APR 协议。
    *   音频拓扑 (Topology) 概念：COPP 与 POPP。
    *   核心调试工具链：QACT, QCAT, QXDM。
    *   标准音频问题排查流程。

3.  **[CAPI 自定义模块开发 (CAPI Module Development)](./03-CAPI-Module-Development.md)**
    *   CAPI vtable 接口与模块生命周期。
    *   process() / set_param() / get_param() 实现详解。
    *   模块注册 (AMDB) 与拓扑集成。
    *   HLOS 侧参数下发与调试技巧。

4.  **[GEF 通用音效框架 (Generic Effects Framework)](./04-GEF-Effects-Framework.md)**
    *   GEF 设计目标与 Use Case 绑定机制。
    *   参数下发/获取 API 与数据流。
    *   GEF 与 ACDB 的关系（默认值 vs 运行时覆写）。
    *   第三方音效集成实践（Dolby / DTS）。

5.  **[低功耗语音唤醒 (LPI Voice Activation)](./05-LPI-Voice-Activation.md)**
    *   LPI 硬件架构与功耗模型。
    *   多级唤醒检测 (Multi-Stage KWD)。
    *   Lab Buffer 回溯机制。
    *   Android SoundTrigger HAL 集成。

6.  **[QACT 调试与工具链 (Debug Tools)](./06-QACT-Debug-Tools.md)**
    *   QACT 实时调参与 ACDB 管理。
    *   QXDM / QCAT 日志抓取与分析。
    *   ADB 音频调试命令大全。
    *   音频问题标准排查流程。

---
[返回主目录](../README.md)
