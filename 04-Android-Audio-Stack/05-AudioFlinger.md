# AudioFlinger 混音引擎深度解析

`AudioFlinger` 是 Android 音频系统的“心脏”，运行在 `audioserver` 进程中。它负责管理所有的音频流、执行软件混音、重采样，并最终通过 HAL 层将数据推送到硬件。

---

## 1. 启动与初始化流程 (Initialization Sequence)

### 1.1 进程启动与实例化
1.  **启动入口**：系统解析 `audioserver.rc`，启动 `/system/bin/audioserver`。
2.  **实例化**：`main_audioserver.cpp` 调用 `AudioFlinger::instantiate()`。
3.  **构造函数**：初始化 `mPlaybackThreads`, `mRecordThreads` 等容器，并创建 `DevicesFactoryHalInterface` 句柄用于 HAL 加载。

### 1.2 HAL 加载链路 (Loading HAL)
当 AudioPolicy 请求加载硬件模块时，链路如下：
`AudioFlinger::loadHwModule_l()` 
-> `DevicesFactoryHalHidl::openDevice()`
-> `DevicesFactory::loadAudioInterface()`
-> **`hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod)`**
*此步骤会根据 `audio.primary.so` 等厂商库名称在系统目录中进行 dlopen 加载。*

---

## 2. 线程模型全景图 (Thread Model)

AudioFlinger 为每个物理输出设备创建一个线程实例。

| 线程类 | 标志 (Flags) | 职责 |
| :--- | :--- | :--- |
| **MixerThread** | `PRIMARY`, `FAST`, `DEEP_BUFFER` | **通用混音**。支持多路流叠加、SRC 和音效处理。 |
| **DirectOutputThread** | `DIRECT` | **直通播放**。跳过 AudioMixer，适用于无损或高位深音频。 |
| **OffloadThread** | `COMPRESS_OFFLOAD` | **硬件卸载**。将压缩流直接推给 DSP 解码，主 CPU 休眠。 |
| **DuplicatingThread** | - | **镜像播放**。将一路流复制到多个输出（如：扬声器与蓝牙同步）。 |

---

## 3. threadLoop 核心循环源码级剖析

所有播放线程的基类逻辑都在 `threadLoop()` 这个死循环中。

### 3.1 处理流程 (Threads.cpp)
```cpp
bool AudioFlinger::PlaybackThread::threadLoop() {
    while (!exitPending()) {
        // 1. 准备阶段 (prepareTracks_l)
        // 🚀 核心：在此步骤中配置 AudioMixer 的各种 Buffer 地址
        // 将每个 Track 的 MainBuffer 指向 PlaybackThread 的 mMixerBuffer
        mMixerStatus = prepareTracks_l(&tracksToRemove);

        // 2. 执行真正的混音 (threadLoop_mix)
        // 内部调用 mAudioMixer->process()，将多个音轨数据合并到 mMixerBuffer
        threadLoop_mix();

        // 3. 音效处理 (EffectChain process)
        // 如果有音效，第一个 EffectChain 的输入会被设置为 mMixerBuffer
        for (auto& chain : mEffectChains) chain->process_l();

        // 4. 写入硬件 (threadLoop_write)
        // 调用 mOutput->write()，最终穿透 HAL 到达 TinyALSA
        threadLoop_write();
    }
}
```

---

## 4. AudioMixer 混音器深度细节

`AudioMixer` 运行在 `libaudioprocessing.so` 中，它是通过 `setParameter` 接口进行配置的。

### 4.1 Buffer 挂载路径
在 `MixerThread::prepareTracks_l` 中：
```cpp
// 将混音器的输出缓冲区设置为线程的临时混音缓冲区
mAudioMixer->setParameter(trackId, AudioMixer::TRACK, AudioMixer::MAIN_BUFFER, (void *)mMixerBuffer);
```

### 4.2 饱和截断算法 (Saturation)
对于每个采样点 $n$，AudioMixer 使用汇编优化的饱和指令：
$Sample_{out} = \text{clamp}(\sum Sample_i \times Gain_i, -32768, 32767)$
这保证了当多路大响度声音叠加时，结果不会产生整数溢出的尖锐杂音（炸音）。

---

## 5. 内存共享机制：Ashmem 与 Proxy

为了实现零拷贝，Android 使用 `AudioTrackServerProxy` 和 `AudioTrackClientProxy` 管理环形缓冲区。

*   **Proxy 的原子操作**：
    *   `obtainBuffer()`：申请一块可写/可读的内存。
    *   `releaseBuffer()`：更新 `sw_ptr` (应用侧) 或 `hw_ptr` (Flinger 侧)。
*   **同步逻辑**：Client 写入数据后更新指针，Server 端在 `threadLoop` 循环中检测到指针变化即开始消费。

---

## 6. 专家调试与 Dump 分析

`adb shell dumpsys media.audio_flinger`
*   **Underruns (UR)**：应用填充太慢。
*   **Frames written**：发送给硬件的总帧数。
*   **Effect Chains**：确认音效链是否挂载成功及其 Buffer 地址。

---
*Next Topic: [AudioPolicy 策略管理深度解析](./06-AudioPolicy.md)*
