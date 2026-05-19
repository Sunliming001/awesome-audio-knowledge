# 10. 音频调试手册 (Audio Debug Cookbook)

本模块提供全链路音频调试的系统方法论与实战 Cookbook，涵盖从 App 层到硬件层的问题定位思路、工具使用与典型案例。

## 📖 章节导航

1.  **[全链路音频调试方法论](./01-Debug-Methodology.md)**
    *   音频数据流全链路拆解与分层定位。
    *   Systrace / Perfetto 音频分析实战。
    *   音频 Tombstone / ANR / Crash 分析。
    *   常见问题 Cookbook：无声、爆音、延迟、功耗、路由异常。

2.  **[跨模块全链路数据流](./02-Cross-Layer-Data-Flow.md)**
    *   场景一：从 App `play()` 到喇叭出声的 7 阶段数据流与延迟分解。
    *   场景二：耳机插入的全栈变化（Hardware → Kernel → Policy → DAPM）。
    *   知识库模块关联地图：串联 01-11 各模块。

3.  **[音频功耗优化 (Power Optimization)](./03-Power-Optimization.md)**
    *   音频全链路功耗模型：AP / DSP / Codec / 接口。
    *   Offload 播放：硬件解码卸载与配置。
    *   AP Suspend 与 Wake Lock 管理。
    *   DSP 低功耗模式：LPI / Clock Scaling。
    *   Codec DAPM 电源管理优化。
    *   功耗调试命令速查与常见问题诊断。

4.  **[音频稳定性分析 (Stability Analysis)](./04-Stability-Analysis.md)**
    *   稳定性基础：信号 (SIGSEGV/SIGABRT)、Tombstone 结构、addr2line 解读。
    *   audioserver Crash：Top 10 场景、分析三步法、调试命令。
    *   Audio HAL Crash：HAL 进程模型、常见 Crash 模式。
    *   ADSP SSR：子系统重启机制、Crash 原因、恢复全链路。
    *   ANR 分析：音频相关 ANR 场景与死锁诊断。
    *   内存问题：UAF / Overflow / Leak、ASan / MTE 调试工具。
    *   稳定性防护：编码防御最佳实践与监控清单。

---
[返回主目录](../README.md)
