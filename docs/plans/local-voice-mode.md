# 语音仅本地处理开关 — 实现计划

## 需求

增加"语音仅本地处理"模式开关。开启后：
- 语音音频不再发送到服务器，不打开音频通道
- 唤醒词检测正常工作，但仅匹配本地命令词时触发本地动作
- UI/LED 显示当前处于"本地模式"
- 通过**本地命令词识别**控制开关（语音说出命令词即可切换）
- 设置持久化到 NVS，重启后保持

## 架构分析

### 当前语音流程

```
MIC → AudioCodec → AudioInputTask → WakeWord/AFE → OpusEncoder → SendQueue
    → application.cc:MAIN_EVENT_SEND_AUDIO → protocol_->SendAudio() → Server
```

触发链路：
1. **唤醒词触发**: `MAIN_EVENT_WAKE_WORD_DETECTED` → `HandleWakeWordDetectedEvent()` → `ContinueWakeWordInvoke()` → `protocol_->OpenAudioChannel()` → `protocol_->SendAudio()`
2. **按键触发**: `MAIN_EVENT_TOGGLE_CHAT` → `HandleToggleChatEvent()` → `protocol_->OpenAudioChannel()`
3. **音频发送**: `application.cc:220-226` — `MAIN_EVENT_SEND_AUDIO` → `protocol_->SendAudio()`

### 本地命令词机制

唤醒词检测返回的是 `GetLastWakeWord()` 字符串（如 "小土豆"、"Hi ESP" 等）。
在 `HandleWakeWordDetectedEvent()` 中，检测到唤醒词后、执行服务器交互前，
先检查 wake word 字符串是否匹配预定义的本地命令词，如果匹配则执行本地动作。

利用已有的 `CustomWakeWord`（multinet）机制，用户需在 `index.json` 中配置本地命令词。

## 本地命令词定义

| 命令词（检测到的 wake word 文本） | 动作 |
|---|---|
| `local_voice_mode_on` | 开启本地语音模式 |
| `local_voice_mode_off` | 关闭本地语音模式 |

用户需要在 `index.json` 中将命令词映射到这些动作词，例如：
```json
{
    "commands": [
        {"command": "本地模式", "text": "local_voice_mode_on", "action": "local"},
        {"command": "退出本地", "text": "local_voice_mode_off", "action": "local"}
    ]
}
```

### CustomWakeWord 改造

在 `custom_wake_word.cc` 中，`action` 字段目前仅支持 `"wake"`。需要增加 `"local"` action 类型，
使本地命令词检测时也能触发回调，但标记为"仅本地，不触发服务器交互"。

改动点（`custom_wake_word.cc`）：
- `Command` 结构不变，已有 `action` 字段
- 在 `AudioDetectionTask()` 中，除 `"wake"` 外增加对 `"local"` action 的处理
- `"local"` action 触发时：设置一个标志 `local_command_ = command.text`，调用 callback
- 新增 `IsLocalCommand()` 和 `GetLocalCommand()` 方法

## 修改文件

### 1. `main/application.h` — 新增成员和接口

```cpp
// 新增:
bool IsLocalVoiceMode() const { return local_voice_mode_; }
void SetLocalVoiceMode(bool enable);

// 新增私有成员:
bool local_voice_mode_ = false;
```

### 2. `main/application.cc` — 核心逻辑修改

#### 2.1 `Initialize()` — 启动时从 NVS 加载设置

在 `audio_service_.Start()` 之后：

```cpp
Settings settings("device", true);
local_voice_mode_ = settings.GetBool("local_voice_mode", false);
```

#### 2.2 `SetLocalVoiceMode()` — 新方法

```cpp
void Application::SetLocalVoiceMode(bool enable) {
    local_voice_mode_ = enable;
    Settings settings("device", true);
    settings.SetBool("local_voice_mode", enable);

    Schedule([this, enable]() {
        auto display = Board::GetInstance().GetDisplay();
        if (enable) {
            if (protocol_ && protocol_->IsAudioChannelOpened()) {
                protocol_->CloseAudioChannel();
            }
            display->SetChatMessage("system", Lang::Strings::LOCAL_VOICE_MODE_ON);
        } else {
            display->SetChatMessage("system", Lang::Strings::LOCAL_VOICE_MODE_OFF);
        }
    });
}
```

#### 2.3 `HandleWakeWordDetectedEvent()` — 增加本地命令词处理

当前逻辑：idle → EncodeWakeWord → ContinueWakeWordInvoke（打开音频通道 + 发送服务器）

新逻辑：

```
HandleWakeWordDetectedEvent():
    1. 获取 wake_word 字符串
    2. 检查是否匹配本地命令词:
       - "local_voice_mode_on"  → SetLocalVoiceMode(true)，return（不触发后续）
       - "local_voice_mode_off" → SetLocalVoiceMode(false)，return（不触发后续）
    3. 如果处于 local_voice_mode_（开启状态）→ 忽略该唤醒词，return
    4. 否则走原有服务器交互流程
```

代码改动：在方法开头增加本地命令词检查：

```cpp
void Application::HandleWakeWordDetectedEvent() {
    auto state = GetDeviceState();
    auto wake_word = audio_service_.GetLastWakeWord();
    ESP_LOGI(TAG, "Wake word detected: %s (state: %d)", wake_word.c_str(), (int)state);

    // === 新增: 本地命令词处理 ===
    if (wake_word == "local_voice_mode_on") {
        SetLocalVoiceMode(true);
        return;
    }
    if (wake_word == "local_voice_mode_off") {
        SetLocalVoiceMode(false);
        return;
    }
    // === 结束 ===

    if (!protocol_) {
        return;
    }
    // ... 原有逻辑 ...

    // 新增: local_voice_mode 下忽略唤醒词
    if (local_voice_mode_) {
        ESP_LOGI(TAG, "Local voice mode enabled, ignoring wake word: %s", wake_word.c_str());
        // 重新启用唤醒词检测，继续监听
        audio_service_.EnableWakeWordDetection(true);
        return;
    }

    // ... 原有 idle/speaking/listening 逻辑不变 ...
}
```

#### 2.4 `HandleToggleChatEvent()` — 拦截按键打开通道

```cpp
void Application::HandleToggleChatEvent() {
    // ... state 检查保持不变 ...

    if (!protocol_) {
        ESP_LOGE(TAG, "Protocol not initialized");
        return;
    }

    if (local_voice_mode_) {
        ESP_LOGI(TAG, "Local voice mode is enabled, cannot toggle chat");
        return;
    }
    // ... 原有逻辑不变
}
```

#### 2.5 `HandleStartListeningEvent()` — 同上

```cpp
void Application::HandleStartListeningEvent() {
    // ... 前面的 state 检查 ...

    if (!protocol_) {
        ESP_LOGE(TAG, "Protocol not initialized");
        return;
    }

    if (local_voice_mode_) {
        ESP_LOGI(TAG, "Local voice mode is enabled, cannot start listening");
        return;
    }
    // ... 原有逻辑不变
}
```

#### 2.6 `WakeWordInvoke()` — 同上拦截

`WakeWordInvoke()` 也有打开音频通道的逻辑，需要增加拦截：

```cpp
void Application::WakeWordInvoke(const std::string& wake_word) {
    if (local_voice_mode_) {
        return;
    }
    // ... 原有逻辑 ...
}
```

#### 2.7 `Run()` 中 `MAIN_EVENT_SEND_AUDIO` — 拦截音频发送

```cpp
if (bits & MAIN_EVENT_SEND_AUDIO) {
    while (auto packet = audio_service_.PopPacketFromSendQueue()) {
        if (local_voice_mode_) {
            break;
        }
        if (protocol_ && !protocol_->SendAudio(std::move(packet))) {
            break;
        }
    }
}
```

### 3. `main/audio/wake_words/custom_wake_word.h` + `custom_wake_word.cc` — 增加 local action

#### custom_wake_word.h

```cpp
// 新增:
bool IsLocalCommand() const { return is_local_command_; }
std::string GetLocalCommand() const { return local_command_; }

// 新增私有成员:
bool is_local_command_ = false;
std::string local_command_;
```

#### custom_wake_word.cc — 在 AudioDetectionTask() 中

当前代码（line ~173-189）：
```cpp
if (mn_state == ESP_MN_STATE_DETECTED) {
    auto& command = commands_[mn_result->command_id[i] - 1];
    if (command.action == "wake") {
        last_detected_wake_word_ = command.text;
        running_ = false;
        wake_word_detected_callback_(last_detected_wake_word_);
    }
}
```

改为：
```cpp
if (mn_state == ESP_MN_STATE_DETECTED) {
    auto& command = commands_[mn_result->command_id[i] - 1];
    if (command.action == "wake" || command.action == "local") {
        last_detected_wake_word_ = command.text;
        if (command.action == "local") {
            is_local_command_ = true;
            local_command_ = command.text;
        } else {
            is_local_command_ = false;
        }
        running_ = false;
        wake_word_detected_callback_(last_detected_wake_word_);
    }
}
```

### 4. `main/boards/common/wifi_board.cc` + `ml307_board.cc` — 设备状态上报

在 `GetDeviceStatusJson()` 的 JSON 对象中增加：

```cpp
cJSON_AddBoolToObject(json, "local_voice_mode", Application::GetInstance().IsLocalVoiceMode());
```

需要 `#include "application.h"`（如果尚未 include）。

### 5. 语言文件 — 增加通知文案

在 `main/assets/locales/zh-CN/language.json` 和 `main/assets/locales/en-US/language.json` 中增加：

**zh-CN:**
```json
"LOCAL_VOICE_MODE_ON": "本地语音模式 已开启",
"LOCAL_VOICE_MODE_OFF": "本地语音模式 已关闭"
```

**en-US:**
```json
"LOCAL_VOICE_MODE_ON": "Local Voice Mode ON",
"LOCAL_VOICE_MODE_OFF": "Local Voice Mode OFF"
```

## 影响范围评估

| 组件 | 影响 | 说明 |
|------|------|------|
| 音频采集 | 无影响 | MIC 仍正常采集，但包被丢弃不发送 |
| 唤醒词检测 | 修改 | 本地模式下仍运行，但仅匹配本地命令词 |
| AfeWakeWord/EspWakeWord | 无影响 | 这两个类不支持本地命令词，仅 CustomWakeWord 支持 |
| 协议层 | 无需修改 | 只在 Application 层拦截即可 |
| 网络 | 无需修改 | WiFi/网络正常连接 |
| 70+ 板子 | 无需修改 | 核心逻辑已覆盖 |

## 实施步骤

1. **application.h** — 添加 `IsLocalVoiceMode()` / `SetLocalVoiceMode()` + `local_voice_mode_`
2. **application.cc** — 添加 `SetLocalVoiceMode()` 实现；`Initialize()` 加载设置；4 个入口点拦截；本地命令词处理
3. **custom_wake_word.h/cc** — 增加 `"local"` action 支持
4. **wifi_board.cc** + **ml307_board.cc** — `GetDeviceStatusJson()` 增加字段
5. **语言文件** — zh-CN 和 en-US 增加通知文案

## 用户配置示例

用户在 `index.json` 中配置本地命令词：

```json
{
    "commands": [
        {
            "command": "本地模式",
            "text": "local_voice_mode_on",
            "action": "local"
        },
        {
            "command": "退出本地",
            "text": "local_voice_mode_off",
            "action": "local"
        },
        {
            "command": "小智你好",
            "text": "Hi 小智",
            "action": "wake"
        }
    ]
}
```

说出 "本地模式" → 开启本地语音模式
说出 "退出本地" → 关闭本地语音模式
说出 "小智你好" → 正常触发服务器对话（仅在非本地模式下）
