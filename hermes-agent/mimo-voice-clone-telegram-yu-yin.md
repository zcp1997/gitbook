---
description: Hermes 接入 Xiaomi MiMo voice clone，并把回复作为 Telegram 语音泡发送的实现步骤。
---

# MiMo Voice Clone 与 Telegram 语音发送

这篇记录 Hermes 里接入 Xiaomi MiMo voice clone TTS，并把最终回复发送成 Telegram 语音泡的完整链路。

适用场景：

* 想把 Hermes 的 `text_to_speech` 工具切到 MiMo。
* 想使用 `mimo-v2.5-tts-voiceclone` 和本地参考音频克隆音色。
* 想让 Telegram 私聊或群聊里的回复自动附带语音泡。

{% hint style="warning" %}
不要把 MiMo API Key、Telegram Bot Token、真实聊天 ID、私有参考音频路径写进公开文档。文档里只放占位符。
{% endhint %}

## 一、整体链路

```text
Telegram 消息
  ↓
GatewayRunner 处理会话
  ↓
AIAgent 生成文本回复
  ↓
Gateway 根据 /voice 模式判断是否需要 TTS
  ↓
tools.tts_tool.text_to_speech_tool
  ↓
MiMo /chat/completions 生成 WAV
  ↓
ffmpeg 转 Ogg/Opus
  ↓
TelegramAdapter.send_voice 发送语音泡
```

关键点：MiMo voice clone 返回的是 base64 音频数据，Hermes 需要先落地为 WAV；Telegram 语音泡要求 Ogg/Opus，所以发送前必须转换。

## 二、准备条件

* Hermes gateway 已接入 Telegram。
* `.env` 中存在 MiMo API Key：
  * `MIMO_API_KEY`，或
  * `XIAOMI_API_KEY`
* 机器上有 `ffmpeg`，用于把 WAV 转成 Ogg/Opus。
* 有一段本地参考音频，例如 MP3 或 WAV。
* gateway 修改配置或代码后，需要重启才会对线上 Telegram 生效。

检查命令：

```bash
which ffmpeg
hermes gateway status
```

## 三、配置 Hermes TTS

在 `~/.hermes/config.yaml` 中配置：

```yaml
tts:
  provider: mimo
  mimo:
    base_url: https://api.xiaomimimo.com/v1
    model: mimo-v2.5-tts-voiceclone
    reference_audio: /path/to/reference-voice.mp3
    format: wav
    style_prompt: "用自然、清晰、温柔但克制的中文女声朗读。语速中等偏稳。"
voice:
  auto_tts: false
```

字段说明：

* `tts.provider: mimo`：让 Hermes 使用内置 MiMo TTS provider。
* `model: mimo-v2.5-tts-voiceclone`：启用 voice clone 模型。
* `reference_audio`：本地参考音频路径。Hermes 会读取文件并转成 `data:audio/...;base64,...` 放进 MiMo 请求的 `audio.voice`。
* `format: wav`：MiMo 稳定返回 WAV，后续再由 Hermes 转成 Telegram 所需的 Opus。
* `style_prompt`：可选朗读风格。如果设为空字符串，MiMo 请求里只发送 assistant 文本。
* `voice.auto_tts: false`：不全局开启所有平台自动语音，改用每个聊天的 `/voice` 模式控制。

{% hint style="info" %}
如果不需要 voice clone，可把 `model` 改成 `mimo-v2.5-tts`，删除 `reference_audio`，并设置 `voice: "冰糖"` 这类预设音色。
{% endhint %}

## 四、MiMo 请求形状

Hermes 的 MiMo provider 使用 Chat Completions endpoint，而不是 OpenAI `/audio/speech` endpoint。

请求要点：

* Endpoint：`POST https://api.xiaomimimo.com/v1/chat/completions`
* Header：`api-key: $MIMO_API_KEY`
* 待合成文本放在 `assistant` message。
* voice clone 时，`audio.voice` 放参考音频 data URI。

示例结构：

```json
{
  "model": "mimo-v2.5-tts-voiceclone",
  "messages": [
    {
      "role": "user",
      "content": "用自然、清晰、温柔但克制的中文女声朗读。语速中等偏稳。"
    },
    {
      "role": "assistant",
      "content": "要合成的正文放这里。"
    }
  ],
  "audio": {
    "format": "wav",
    "voice": "data:audio/mpeg;base64,<REFERENCE_AUDIO_BASE64>"
  }
}
```

响应里取音频：

```python
audio_b64 = data["choices"][0]["message"]["audio"]["data"]
audio_bytes = base64.b64decode(audio_b64)
```

## 五、代码接入点

Hermes 里主要涉及两个位置。

### 1. `tools/tts_tool.py`

MiMo provider 的核心逻辑：

* 读取 `MIMO_API_KEY` 或 `XIAOMI_API_KEY`。
* 读取 `tts.mimo` 配置。
* 如果存在 `reference_audio`：
  * 校验文件存在；
  * 判断 MIME 类型；
  * base64 编码；
  * 覆盖 `audio.voice` 为 data URI。
* 调用 MiMo `/chat/completions`。
* 从 `choices[0].message.audio.data` 解码音频。
* 先写出 WAV。

Telegram 环境下，`text_to_speech_tool` 会把 MiMo 生成的 WAV 继续转成 Ogg/Opus，并在返回 JSON 中标记：

```text
[[audio_as_voice]]
MEDIA:/absolute/path/to/audio.ogg
```

### 2. `gateway/run.py`

Gateway 自动语音回复链路：

* `_load_voice_modes()` 从 `~/.hermes/gateway_voice_mode.json` 读取每个聊天的语音模式。
* `_should_send_voice_reply()` 判断是否需要自动发送语音：
  * `off`：不发；
  * `voice_only`：仅语音输入触发语音回复；
  * `all`：文本和语音输入都触发语音回复。
* `_send_voice_reply()`：
  * 清理 Markdown 文本；
  * 调用 `text_to_speech_tool()`；
  * 读取返回的真实 `file_path`；
  * 如果 Telegram 收到的不是 `.ogg` 或 `.opus`，再做一次防御性 Opus 转换；
  * 调用 `adapter.send_voice()` 发送语音泡；
  * 最后清理临时文件。

## 六、开启 Telegram 自动语音

在目标 Telegram 聊天里发送：

```text
/voice tts
```

效果：该聊天里的每次 Hermes 最终文本回复都会附一条语音泡。

其他模式：

* `/voice on`：通常用于 voice-to-voice，只在语音输入后回复语音。
* `/voice off`：关闭该聊天的自动语音。

持久化文件形如：

```json
{
  "telegram:<CHAT_ID>": "all"
}
```

`<CHAT_ID>` 使用真实聊天 ID，但不要写进公开文档。

## 七、重启与验证

配置或代码改完后，重启 gateway：

```bash
hermes gateway restart
hermes gateway status
```

基础验证：

```bash
python -m py_compile tools/tts_tool.py gateway/run.py
python -m pytest -q tests/tools/test_tts_mimo.py tests/gateway/test_voice_command.py::TestHandleVoiceCommand tests/gateway/test_voice_command.py::TestAutoVoiceReply tests/gateway/test_voice_command.py::TestSendVoiceReply
```

手动验证：

1. 在 Telegram 目标聊天发送 `/voice tts`。
2. 发送一句普通文本给 Hermes。
3. 确认先后收到：
   * 文本回复；
   * Telegram 原生 voice bubble，而不是文件附件。
4. 查看 gateway 日志是否有 TTS 或 Opus 转换错误：

```bash
rg -n "Auto voice reply|MiMo TTS|Opus|send_voice|TTS generation failed" ~/.hermes/logs/gateway.log
```

## 八、常见问题

### 收到的是文件，不是语音泡

通常是最终文件不是 Ogg/Opus。

处理：

* 确认安装 `ffmpeg`。
* 确认 `_send_voice_reply()` 里有 Telegram 防御性转换。
* 确认 `text_to_speech_tool()` 返回的 `file_path` 是 `.ogg` 或 `.opus`。

### MiMo 返回 401

API Key 错误或没有被 gateway 进程加载。

处理：

* 检查 `.env` 中是否有 `MIMO_API_KEY` 或 `XIAOMI_API_KEY`。
* 修改 `.env` 后重启 gateway。
* 不要把 key 写进 `config.yaml` 或文档。

### reference audio 找不到

`reference_audio` 是 gateway 进程所在机器能访问的本地路径。路径不存在时，Hermes 会报：

```text
MiMo reference_audio not found
```

处理：

* 使用绝对路径。
* 确认 gateway 运行用户有读权限。
* 如果是多 profile，确认配置写在当前 profile 对应的 `config.yaml`。

### 改了代码但 Telegram 没变化

磁盘代码已修改不代表线上 gateway 已生效。

处理：

```bash
hermes gateway restart
hermes gateway status
```

然后重新在 Telegram 里发消息验证。

## 九、最小验收标准

一次接入完成后，至少满足：

* `text_to_speech_tool()` 使用 MiMo voice clone 成功生成音频。
* Telegram 自动语音回复最终发送的是 Ogg/Opus voice bubble。
* `/voice tts`、`/voice on`、`/voice off` 行为符合预期。
* gateway 重启后配置仍然生效。
* 日志里没有泄露 API Key、Bot Token、真实聊天 ID 或参考音频 base64。
