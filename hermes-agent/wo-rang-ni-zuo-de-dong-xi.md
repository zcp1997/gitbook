---
description: 如何使用工具提高效率
---

# 我用 hermes 做了什么

## 自动化与脚本

* 一点万象自动签到：基于抓包参数的签到脚本。
* V2EX 自动签到：通过浏览器会话完成每日领取。
* RSS 个性化推荐：按主人偏好筛选技术、产品、AI、互联网内容，可附语音播报。
* 模型 Provider 余额查询：集中查询常用模型服务余额，凭据独立存放。
* 低磁盘空间告警：本机磁盘余量不足时提醒。
* 家宽公网 IP 段台账：记录家宽动态 IP 归属段，便于后续白名单维护。
* Hermes 全量备份：定期把 `.hermes` 状态备份到 iCloud。
* 数字资产台账检查：维护订阅、服务、到期日和月度消费报告。

## Hermes 能力扩展

* Surge CLI skill：通过 Surge CLI 查询环境、策略、请求、设备和连接状态。
* 小红书视频下载流程：优先保存原始可用视频变体，并保留原始资源包。
* 百度网盘 skill：管理百度网盘文件、分享、下载和登录状态。
* MiMo 语音回复：Telegram 里使用 MiMo 语音生成伊莉娜风格语音泡，详见 [MiMo Voice Clone 与 Telegram 语音发送](mimo-voice-clone-telegram-yu-yin.md)。
* Telegram 大视频发送：超过 `50 MB`尽量通过本地 Bot API 保留原始质量。

## 本机服务配置

* Watchtower 容器监测：用于指定容器的自动镜像更新检查。
* Hermes memory 调整：使用内置记忆提供者，减少外部依赖。
* Surge 请求排查：可按固定设备 IP 辅助定位 Apple TV、手机等设备的连接。

## 实用工具

* 漫画图片转 PDF：把下载目录中的图片按顺序合成为 PDF。
* GitBook 文档维护：通过本地 Git Sync 仓库编辑、提交并推送文档。
