# 基于 MCP 协议的 AI 聊天机器人

## 项目简介

本项目基于小智 AI 开源项目二次开发，以 ESP32-S3 为核心，结合大语言模型（Qwen / DeepSeek 等），实现语音交互与多端控制。

目标硬件平台：**Waveshare ESP32-S3-Touch-AMOLED-1.8**

## 已实现功能

- Wi-Fi 联网与 Blufi 蓝牙配网
- 离线语音唤醒（ESP-SR）
- 基于流式 ASR + LLM + TTS 的语音交互
- 支持 Websocket / MQTT+UDP 通信协议
- OPUS 音频编解码
- 1.8 寸 AMOLED 触摸屏显示（表情 + 状态栏）
- 电量显示与电源管理（AXP2101）
- 长按 Boot 按钮清除 WiFi 配置并重启
- 通过 MCP 协议实现设备控制（音量、亮度等）
- 语音调节音量（对设备说"音量调到XX"）

## 硬件配置

| 项目 | 规格 |
|------|------|
| 主控 | ESP32-S3 |
| 显示屏 | 1.8 寸 AMOLED（SH8601），368x448 |
| 触摸 | FT5x06 电容触摸 |
| 音频编解码 | ES8311 |
| 电源管理 | AXP2101 |
| 音频采样率 | 输入 24000Hz / 输出 24000Hz |

## 服务器配置

### OTA 服务器地址

OTA 服务器用于固件版本检查和获取通信服务器地址。配置方式：

**方式一：编译时配置（Kconfig）**

在 `menuconfig` 中修改 `Xiaozhi Assistant` -> `Default OTA URL`，默认值为：
```
https://api.tenclass.net/xiaozhi/ota/
```

如需使用私有服务器，修改 `main/Kconfig.projbuild` 中的 `OTA_URL`：
```
config OTA_URL
    string "Default OTA URL"
    default "http://你的服务器IP:端口/xiaozhi/ota/"
```

**方式二：运行时通过 NVS 配置**

设备启动后会从 NVS `wifi` 命名空间读取 `ota_url`，如果存在则优先使用。

### 通信服务器

设备通过 OTA 接口获取 WebSocket 或 MQTT 服务器地址，自动存入 NVS：

- **WebSocket 模式：** 地址存于 NVS，由 OTA 服务器下发
- **MQTT 模式：** `endpoint`、`client_id`、`username`、`password` 等参数存于 NVS `mqtt` 命名空间

### Blufi 蓝牙配网

设备蓝牙广播名称为 `turing-Blufi`，可通过蓝牙配网工具连接并配置 WiFi。

## 开发环境

- VSCode + ESP-IDF 插件
- ESP-IDF SDK 版本 5.4 及以上
- 目标芯片：ESP32-S3

## 编译与烧录

1. 克隆项目并初始化子模块
2. 在 ESP-IDF 中选择目标芯片为 ESP32-S3
3. 在 `menuconfig` 中选择 Board Type 为 `Waveshare ESP32-S3-Touch-AMOLED-1.8`
4. 如需连接私有服务器，修改 `menuconfig` 中的 `Default OTA URL`
5. 编译并烧录

## Bug 修复记录

### 修复：非首句回答语速过快

**问题：** 第一句回答语速正常，后续回答语速明显加快。

**原因：** `ResetDecoder()` 中重置了 opus 解码器，但未重置输出重采样器（`output_resampler_`）。重采样器内部滤波器状态残留，导致后续音频重采样输出样本数异常。

**修复文件：** `main/audio/audio_service.cc`

在 `ResetDecoder()` 中添加 `esp_ae_rate_cvt_reset(output_resampler_)` 调用。

### 优化：Blufi WiFi 扫描逻辑

重构了 WiFi 扫描流程，修复 AP 模式下扫描失败的问题，统一了扫描代码路径。

**修复文件：** `main/boards/common/blufi.cpp`

### 增强：长按清除 WiFi 配置

为 Waveshare ESP32-S3-Touch-AMOLED-1.8 板添加长按 Boot 按钮（4秒）清除 WiFi 配置并自动重启的功能。

**修复文件：** `main/boards/esp32-s3-touch-amoled-1.8/esp32-s3-touch-amoled-1.8.cc`

## 项目结构

```
main/
  ├── audio/                    # 音频处理（编解码、重采样）
  │   ├── audio_service.cc      # 音频服务核心
  │   └── codecs/               # 各编解码器驱动（ES8311 等）
  ├── boards/                   # 硬件板级支持
  │   ├── esp32-s3-touch-amoled-1.8/  # 目标板配置与初始化
  │   └── common/               # 公共组件（Blufi 配网等）
  ├── protocols/                # 通信协议
  │   ├── websocket_protocol.cc # WebSocket 协议实现
  │   └── mqtt_protocol.cc      # MQTT 协议实现
  ├── ota.cc                    # OTA 固件升级
  └── application.cc            # 应用主逻辑与状态机
```

## 致谢

本项目基于 [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) 开源项目，采用 MIT 许可证。
