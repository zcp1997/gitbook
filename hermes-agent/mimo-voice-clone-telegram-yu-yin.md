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
  ↓
ffprobe 读取 Ogg/Opus 时长，并把 duration 传给 Telegram
```

关键点：MiMo voice clone 返回的是 base64 音频数据，Hermes 需要先落地为 WAV；Telegram 语音泡要求 Ogg/Opus，所以发送前必须转换。发送前再用 `ffprobe` 读取音频时长并传给 Telegram，可以让语音泡显示时长更稳定。

## 二、准备条件

* Hermes gateway 已接入 Telegram。
* `.env` 中存在 MiMo API Key：
  * `MIMO_API_KEY`，或
  * `XIAOMI_API_KEY`
* 机器上有 `ffmpeg`，用于把 WAV 转成 Ogg/Opus。
* 机器上有 `ffprobe`，用于读取 Ogg/Opus duration。
* 有一段本地参考音频，例如 MP3 或 WAV。
* gateway 修改配置或代码后，需要重启才会对线上 Telegram 生效。

检查命令：

```bash
which ffmpeg
which ffprobe
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

Hermes 里主要涉及三个位置。

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

{% hint style="info" %}
如果最终文本里大部分是代码、英文日志、JSON、堆栈或命令输出，直接全文转语音的体验通常很差。更合理的策略是：文字回复保持完整；语音只读一段中文摘要，并在文字里标明“语音是摘要版”。
{% endhint %}

### 3. `gateway/platforms/telegram.py`

Telegram 原生语音泡的发送逻辑：

* `.ogg` / `.opus` 走 `bot.send_voice()`，显示为 Telegram voice bubble。
* `.mp3` / `.m4a` 走 `bot.send_audio()`，显示为音频文件。
* `.wav` / `.flac` 等 Telegram 不能作为 voice bubble 播放的格式，回退为文档发送。
* 发送 `.ogg` / `.opus` 前调用 `_probe_local_audio_duration()`：
  * 使用 `ffprobe -show_entries format=duration -of json <audio_path>` 读取时长；
  * 用 `int(round(duration))` 转成秒；
  * 成功时把 `duration` 传给 `bot.send_voice()`；
  * 失败时不阻断发送，只是不传 duration。

对应测试文件：

```text
tests/gateway/test_telegram_voice_duration.py
```

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
python -m py_compile tools/tts_tool.py gateway/run.py gateway/platforms/telegram.py
python -m pytest -q tests/tools/test_tts_mimo.py tests/gateway/test_voice_command.py::TestHandleVoiceCommand tests/gateway/test_voice_command.py::TestAutoVoiceReply tests/gateway/test_voice_command.py::TestSendVoiceReply
python -m pytest -q tests/gateway/test_telegram_voice_duration.py
```

手动验证：

1. 在 Telegram 目标聊天发送 `/voice tts`。
2. 发送一句普通文本给 Hermes。
3. 确认先后收到：
   * 文本回复；
   * Telegram 原生 voice bubble，而不是文件附件。
4. 确认语音泡显示时长基本正确。
5. 查看 gateway 日志是否有 TTS、Opus 转换或 duration 探测错误：

```bash
rg -n "Auto voice reply|MiMo TTS|Opus|send_voice|ffprobe|duration|TTS generation failed" ~/.hermes/logs/gateway.log
```

## 八、语音内容策略与调优

自动语音不应该机械朗读最终文本的全部内容。Telegram 上最常见的问题是：最终回复里有大量代码、英文日志、路径、JSON 或命令输出，TTS 读出来既长又难听清，用户体验反而变差。

推荐策略：

* **文字回复保留完整内容**：代码、命令、错误堆栈、路径和检查结果仍然放在文字里。
* **语音回复使用摘要版**：只读结论、关键动作、风险和下一步。
* **明确标注**：如果语音不是全文，文字里写清楚“语音是摘要版”。
* **摘要用中文自然语言**：避免逐字读英文函数名、长路径、JSON key、日志噪声。
* **短于 60 秒优先**：语音泡适合听结论，不适合听完整技术报告。

可用的调优规则：

1. 如果最终文本长度较短，且没有大段代码或日志，可以全文转语音。
2. 如果包含代码块、长英文段落、终端输出、JSON、diff、表格，应生成摘要再转语音。
3. 摘要结构建议固定为：
   * 结论；
   * 已做动作；
   * 验证结果；
   * 需要用户知道的风险或下一步。
4. 对技术名词保留必要英文，但不要朗读大段代码。
5. 如果语音是摘要版，最终文字回复应显式标注，避免用户以为语音被截断。

摘要示例：

```text
语音是摘要版：已完成 MiMo voice clone 到 Telegram 语音泡的链路；
发送前会把 WAV 转成 Ogg/Opus，并用 ffprobe 读取时长传给 Telegram；
相关测试已通过。完整代码和验证命令见文字部分。
```

### 调优检查清单

每次调整 TTS 体验后，按这个顺序验收：

1. **格式**：最终发送的是 Telegram voice bubble，不是普通文件。
2. **时长**：语音泡显示时长与实际播放时长基本一致。
3. **可听性**：不朗读大段代码、英文日志或 JSON。
4. **一致性**：语音摘要和文字回复不矛盾。
5. **标注**：摘要版语音要在文字里说明。
6. **回退**：`ffprobe` 或 duration 探测失败时，不能影响语音发送。
7. **重启**：代码或配置修改后，确认 gateway 已重启并加载新代码。

## 九、常见问题

### 收到的是文件，不是语音泡

通常是最终文件不是 Ogg/Opus。

处理：

* 确认安装 `ffmpeg`。
* 确认 `_send_voice_reply()` 里有 Telegram 防御性转换。
* 确认 `text_to_speech_tool()` 返回的 `file_path` 是 `.ogg` 或 `.opus`。

### 语音泡时长不准

通常是发送时没有给 Telegram 传 duration，或本地 `ffprobe` 无法读取文件时长。

处理：

* 确认安装 `ffprobe`。
* 用本地命令检查音频文件：

```bash
ffprobe -v error -show_entries format=duration -of json /path/to/audio.ogg
```

* 确认 `TelegramAdapter.send_voice()` 对 `.ogg` / `.opus` 调用了 `_probe_local_audio_duration()`。
* 确认相关测试通过：

```bash
python -m pytest -q tests/gateway/test_telegram_voice_duration.py
```

### 语音太长、代码和英文太多，听不清楚

这是内容策略问题，不是单纯的 TTS provider 问题。

处理：

* 文字回复保留完整技术细节。
* 语音只合成中文摘要版。
* 在最终文字里标注“语音是摘要版”。
* 摘要只保留结论、已做动作、验证结果和下一步。

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

## 十、最小验收标准

一次接入完成后，至少满足：

* `text_to_speech_tool()` 使用 MiMo voice clone 成功生成音频。
* Telegram 自动语音回复最终发送的是 Ogg/Opus voice bubble。
* `/voice tts`、`/voice on`、`/voice off` 行为符合预期。
* `.ogg` / `.opus` 发送前会尽量探测 duration，并传给 Telegram。
* 大段代码、英文日志或 JSON 场景下，语音使用摘要版，而不是机械朗读全文。
* gateway 重启后配置仍然生效。
* 日志里没有泄露 API Key、Bot Token、真实聊天 ID 或参考音频 base64。
