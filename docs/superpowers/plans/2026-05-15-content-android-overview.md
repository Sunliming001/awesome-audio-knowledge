# Android Audio Stack Overview Content Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a high-quality technical document `01-Overview.md` in the `04-Android-Audio-Stack` directory.

**Architecture:** Classic Technical Document Structure.

**Tech Stack:** Markdown, Mermaid.

---

### Task 1: Write 01-Android-Audio-Stack/01-Overview.md

**Files:**
- Create: `04-Android-Audio-Stack/01-Overview.md`

- [ ] **Step 1: Write the content of 01-Overview.md**

Write the following content to `04-Android-Audio-Stack/01-Overview.md`:

```markdown
# Android 音频系统概览 (Android Audio Stack Overview)

Android 音频系统是 Android 框架中最复杂、最深奥的子系统之一。它负责从应用层到底层硬件的音频流管理、路由决策以及音频处理。

---

## 1. 整体架构 (High-Level Architecture)

Android 音频栈遵循典型的分层架构，从上到下可分为五个主要层级：

```mermaid
graph TD
    subgraph "App Layer"
        App[Music/Communication App]
        API[AudioTrack / AudioRecord / MediaPlayer]
    end

    subgraph "Framework Layer (Java)"
        AS[AudioService]
        AM[AudioManager]
    end

    subgraph "Native Service Layer (C++)"
        AF[AudioFlinger - 混音与播放]
        APS[AudioPolicyService - 策略与路由]
    end

    subgraph "HAL Layer (Hardware Abstraction)"
        HAL[Audio HAL - HIDL / AIDL]
    end

    subgraph "Kernel Layer"
        ALSA[ALSA / ASoC Driver]
        Hardware[Audio DSP / Codec]
    end

    App --> API
    API --> AS
    AS -- Binder --> APS
    API -- JNI/Binder --> AF
    APS --> AF
    AF --> HAL
    HAL --> ALSA
    ALSA --> Hardware
```

---

## 2. 各层级核心职责

### 2.1 应用层 (App Layer)
开发者使用 Java/Kotlin API（如 `AudioTrack` 用于原始 PCM 播放，`AudioRecord` 用于录音）或更高级的接口（如 `ExoPlayer`, `Oboe`）。

### 2.2 Framework 层 (Java)
*   **AudioService**：负责系统音量控制、音频焦点 (Audio Focus) 管理。
*   **AudioManager**：App 访问音频系统的主要入口。

### 2.3 Native 服务层 (C++)
这是 Android 音频的心脏，包含两个关键进程（通常运行在 `audioserver` 进程中）：
*   **AudioFlinger**：音频引擎。负责音频流的混音 (Mixing)、重采样 (Resampling)、音量调节以及将数据写入 HAL。
*   **AudioPolicyService**：策略引擎。负责决定“音频从哪里播放”（如：耳机拔出后自动切换到扬声器）。

### 2.4 HAL 层 (Hardware Abstraction Layer)
将通用的音频服务与具体的硬件驱动解耦。定义了 `IDevice` 和 `IStream` 等接口。

### 2.5 Kernel 层 (内核)
主要基于 **ALSA (Advanced Linux Sound Architecture)**。ASoC (ALSA System on Chip) 负责 SoC 内部 DSP 与外部 Codec 的驱动交互。

---

## 3. 音频流的关键路径

1.  **控制流 (Control Flow)**：设置音量、切换路由。主要由 `AudioPolicyService` 处理。
2.  **数据流 (Data Flow)**：音频 PCM 数据的传输。主要由 `AudioFlinger` 和共享内存 (Shared Memory) 处理。

---

## 4. 关键参考 (References)

1.  [Android Open Source Project - Audio](https://source.android.com/devices/audio)
2.  [Embedded Android - Karim Yaghmour](https://www.oreilly.com/library/view/embedded-android/9781449327958/)

---
*Next Topic: [AudioTrack 深度解析](./02-AudioTrack-Deep-Dive.md)*
```

- [ ] **Step 2: Commit the file**

Run:
```bash
git add 04-Android-Audio-Stack/01-Overview.md
git commit -m "feat: add Android audio stack overview"
```

---
End of plan.
