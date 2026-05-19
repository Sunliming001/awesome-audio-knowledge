# Android 音频 CTS/VTS 与自动化测试

Android 设备出厂前必须通过 Google 的 CTS (Compatibility Test Suite) 和 VTS (Vendor Test Suite) 音频测试，确保 API 行为一致性和 HAL 接口合规。本章详解音频相关的 CTS/VTS 用例及自动化测试框架搭建。

---

## 1. CTS 音频测试概览

```
CTS (Compatibility Test Suite) 音频模块:

  测试目标: 验证 Android SDK 音频 API 行为是否符合 CDD (Compatibility Definition)
  
  核心测试包:
  ┌────────────────────────────────────────────────────────────────┐
  │ 模块                          测试用例数  覆盖范围            │
  ├────────────────────────────────────────────────────────────────┤
  │ CtsMediaAudioTestCases        ~200+      AudioTrack/Record   │
  │                                           AudioManager        │
  │                                           AudioEffect         │
  │                                           AudioAttributes     │
  ├────────────────────────────────────────────────────────────────┤
  │ CtsAudioTestCases             ~150+      AudioFocus          │
  │                                           音量控制            │
  │                                           设备路由            │
  ├────────────────────────────────────────────────────────────────┤
  │ CtsMediaMiscTestCases         ~50+       MediaPlayer 音频    │
  │                                           MediaCodec 音频     │
  ├────────────────────────────────────────────────────────────────┤
  │ CtsAudioHostTestCases         ~30+       USB Audio           │
  │                                           BT Audio            │
  │                                           MMAP 路径           │
  └────────────────────────────────────────────────────────────────┘
```

### 1.1 关键 CTS 音频测试用例

```
高频失败的 CTS 音频用例:

  1. AudioTrackTest#testPlaybackRate
     → 验证 setPlaybackRate 是否在 0.5x ~ 2.0x 范围正常工作
     → 失败原因: 某些 HAL 不支持变速播放
     
  2. AudioRecordTest#testRecordingFromSource
     → 验证各种 AudioSource (MIC, VOICE_RECOGNITION, UNPROCESSED)
     → 失败原因: UNPROCESSED 源仍有 NS/AEC 处理
     
  3. AudioEffectTest#testEqualizerProperties
     → 验证系统 EQ 音效的 band 数量和频率范围
     → 失败原因: 音效库未正确注册
     
  4. AudioManagerTest#testMusicActiveCount
     → 验证 isMusicActive() API 正确性
     → 失败原因: 并发流计数不准确
     
  5. SpatializerTest (Android 13+)
     → 验证 Spatializer API 可用性
     → 失败原因: 未实现 Spatializer Effect
```

### 1.2 运行 CTS 音频测试

```bash
# === CTS 测试执行 ===

# 1. 下载 CTS 工具包
# https://source.android.com/docs/compatibility/cts/downloads

# 2. 连接设备, 运行全部音频测试
./cts-tradefed run cts -m CtsMediaAudioTestCases
./cts-tradefed run cts -m CtsAudioTestCases

# 3. 运行单个测试
./cts-tradefed run cts -m CtsMediaAudioTestCases \
    -t android.media.cts.AudioTrackTest#testPlaybackRate

# 4. 查看测试结果
# 结果输出: out/host/linux-x86/cts/results/
# HTML 报告: test_result.html
# XML 详情: test_result.xml

# === 常用调试 ===
# 测试时同时抓 logcat:
adb logcat -v time > cts_audio_log.txt &
./cts-tradefed run cts -m CtsMediaAudioTestCases

# 筛选音频相关日志:
grep -iE "AudioTrack|AudioRecord|AudioFlinger|AudioPolicy" cts_audio_log.txt
```

---

## 2. VTS 音频测试

```
VTS (Vendor Test Suite) 音频模块:

  测试目标: 验证 Audio HAL 实现是否符合 AIDL/HIDL 接口规范
  
  核心测试:
  ┌──────────────────────────────────────────────────────────────┐
  │ VtsHalAudioCoreTargetTest                                   │
  │   → IModule: 打开/关闭模块, 获取端口信息                    │
  │   → IStreamOut: 写入 PCM, 检查时间戳, drain/flush          │
  │   → IStreamIn: 读取 PCM, 检查时间戳                        │
  │   → 格式支持: 验证 HAL 报告的 format/rate/channel          │
  │                                                              │
  │ VtsHalAudioEffectTargetTest                                 │
  │   → IEffect: 音效加载/卸载, 参数设置/获取                   │
  │   → 预置音效: Equalizer, BassBoost, Virtualizer            │
  │                                                              │
  │ VtsHalSoundTriggerTargetTest                                │
  │   → ISoundTriggerHw: 模型加载, 识别启动/停止               │
  └──────────────────────────────────────────────────────────────┘
```

### 2.1 运行 VTS 音频测试

```bash
# VTS 测试执行
./vts-tradefed run vts -m VtsHalAudioCoreTargetTest

# 单个测试
./vts-tradefed run vts -m VtsHalAudioCoreTargetTest \
    -t AudioCoreModuleTest.OpenPrimaryDevice

# VTS 常见失败:
# 1. "IStreamOut::write timeout"
#    → HAL write 超时, 检查 DMA/ALSA 配置
# 2. "Unexpected format"
#    → HAL 报告的 supportedFormats 与实际不匹配
# 3. "Timestamp not advancing"
#    → getPresentationPosition 返回值未更新
```

---

## 3. 音频自动化测试框架

### 3.1 基于 Python 的自动化测试

```python
#!/usr/bin/env python3
"""
Android 音频自动化测试框架示例
依赖: adb, numpy, scipy
"""

import subprocess
import numpy as np
from scipy.io import wavfile
import time

class AudioAutoTest:
    def __init__(self, device_serial=None):
        self.adb_prefix = ["adb"]
        if device_serial:
            self.adb_prefix += ["-s", device_serial]
    
    def adb_shell(self, cmd):
        """执行 adb shell 命令"""
        result = subprocess.run(
            self.adb_prefix + ["shell"] + cmd.split(),
            capture_output=True, text=True, timeout=30
        )
        return result.stdout.strip()
    
    def play_tone(self, freq_hz=1000, duration_s=3, sample_rate=48000):
        """生成并播放正弦波测试音"""
        t = np.linspace(0, duration_s, sample_rate * duration_s)
        tone = (np.sin(2 * np.pi * freq_hz * t) * 32767).astype(np.int16)
        wavfile.write("/tmp/test_tone.wav", sample_rate, tone)
        
        subprocess.run(self.adb_prefix + ["push", "/tmp/test_tone.wav",
                       "/data/local/tmp/test_tone.wav"])
        self.adb_shell("am start -a android.intent.action.VIEW "
                       "-d file:///data/local/tmp/test_tone.wav "
                       "-t audio/wav")
    
    def record_audio(self, duration_s=3, sample_rate=48000, channels=1):
        """录音并拉回本地"""
        self.adb_shell(f"tinycap /data/local/tmp/rec.wav "
                       f"-D 0 -d 0 -c {channels} -r {sample_rate} "
                       f"-b 16 -T {duration_s}")
        subprocess.run(self.adb_prefix + ["pull",
                       "/data/local/tmp/rec.wav", "/tmp/rec.wav"])
        return wavfile.read("/tmp/rec.wav")
    
    def check_audio_active(self):
        """检查是否有活跃的音频流"""
        dump = self.adb_shell("dumpsys media.audio_flinger")
        return "ACTIVE" in dump
    
    def get_volume(self, stream="music"):
        """获取当前音量"""
        dump = self.adb_shell("dumpsys audio")
        # 解析音量信息...
        return dump
    
    def verify_routing(self, expected_device):
        """验证音频路由"""
        dump = self.adb_shell("dumpsys media.audio_policy")
        return expected_device in dump

# 使用示例:
# test = AudioAutoTest()
# test.play_tone(1000, 3)
# sr, data = test.record_audio(3)
# assert test.check_audio_active()
```

### 3.2 自动化测试 CI 集成

```
音频自动化测试 CI/CD 流程:

  ┌─────────────────────────────────────────────────────────┐
  │ 代码提交 (Git Push)                                    │
  │   │                                                     │
  │   ▼                                                     │
  │ Jenkins / GitLab CI 触发                                │
  │   │                                                     │
  │   ├── 编译固件 (make -j)                               │
  │   ├── 刷机 (fastboot flash)                            │
  │   ├── 等待设备启动 (adb wait-for-device)               │
  │   │                                                     │
  │   ├── 运行 CTS 音频子集                                │
  │   │   ./cts-tradefed run cts -m CtsMediaAudioTestCases │
  │   │                                                     │
  │   ├── 运行 VTS 音频测试                                │
  │   │   ./vts-tradefed run vts -m VtsHalAudioCoreTarget  │
  │   │                                                     │
  │   ├── 运行自定义音频回归测试                           │
  │   │   python3 audio_regression_test.py                  │
  │   │     - 播放/录音功能验证                            │
  │   │     - 音量控制验证                                 │
  │   │     - 路由切换验证 (SPK/HP/BT)                     │
  │   │     - 通话音频验证                                 │
  │   │                                                     │
  │   ├── 生成测试报告 (HTML/JUnit XML)                    │
  │   └── 邮件/钉钉通知结果                               │
  └─────────────────────────────────────────────────────────┘
  
  关键指标:
    - CTS 通过率目标: 100%
    - VTS 通过率目标: 100%
    - 自定义测试通过率: > 95% (允许环境因素导致的 flaky)
    - 测试执行时间: < 30 分钟 (音频子集)
```

---

## 4. 音频压力测试

```bash
# === 音频压力测试脚本 ===

# 1. 播放/暂停循环 (测试资源泄漏)
for i in $(seq 1 1000); do
    adb shell am start -a android.intent.action.VIEW \
        -d file:///system/media/audio/ringtones/Ring_Synth_04.ogg \
        -t audio/ogg
    sleep 1
    adb shell am force-stop com.android.music
    echo "Iteration $i done"
done

# 2. 并发流测试
# 同时播放多个 AudioTrack, 验证混音是否正常
adb shell /data/local/tmp/audio_stress_test --streams 8 --duration 60

# 3. 路由切换压力测试
# 快速插拔耳机 / 连断蓝牙, 检查是否有无声/crash
for i in $(seq 1 200); do
    adb shell input keyevent KEYCODE_HEADSETHOOK
    sleep 0.5
done

# 4. 监控异常
# 同时运行:
adb logcat -v time | grep -iE "crash|fatal|anr|tombstone|audio" &
adb shell dumpsys meminfo audioserver | tail -5  # 内存泄漏检查
```

---

## 5. 关键参考 (References)

1. [Android CTS Downloads](https://source.android.com/docs/compatibility/cts/downloads)
2. [Android CDD - Audio Section](https://source.android.com/docs/compatibility/cdd)
3. [Android VTS Documentation](https://source.android.com/docs/core/tests/vts)
4. [Audio HAL AIDL Interface Tests](https://cs.android.com/android/platform/superproject/+/master:hardware/interfaces/audio/)
5. [Tradefed Test Framework](https://source.android.com/docs/core/tests/tradefed)
