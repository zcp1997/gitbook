---
description: 用 Hermes Agent 自动整理 Telegram 在花频道日报的实现思路
icon: newspaper
---

# 解析 Telegram 在花频道的核心思路

这页记录一个 Telegram 频道日报的实现方式：每天读取频道新消息、评论区反馈和 reaction 数据，再交给 Hermes Agent 生成一份能直接阅读的中文摘要。

目标不是把频道消息机械搬运一遍，而是回答三个问题：

1. 今天频道里哪些内容最受关注？
2. 评论区主要在争论什么、补充什么？
3. 哪些高反应评论值得单独保留链接，方便回看原文？

## 为什么不用普通 Bot API

普通 Telegram Bot API 更适合“机器人在自己的聊天里收发消息”。如果要回看一个频道的历史消息、关联评论、reaction 和讨论串，它的限制会比较多。

这里采用 MTProto 用户会话，也就是用一个已登录的 Telegram 账号读取自己可见的频道内容。这样可以更稳定地拿到：

- 频道消息正文与发布时间；
- 每条消息的 reaction 统计；
- 关联讨论区里的评论；
- 评论自己的 reaction 统计；
- 原始消息和评论的跳转链接。

## 整体流程

核心流程可以分成六步：

1. 设定统计窗口，例如每天从前一天 06:00 到当天 06:00。
2. 用 Telethon 打开 Telegram 会话并定位频道。
3. 遍历窗口内的频道消息，提取正文、发布时间、评论数和 reaction。
4. 对有评论的消息继续读取讨论区评论，记录评论正文、reaction、媒体类型和跳转链接。
5. 按规则过滤噪声评论，分别生成“频道消息热度榜”和“评论区热评榜”。
6. 把结构化结果交给 Hermes Agent，由模型写成自然语言日报。

这套设计把“采集”和“写摘要”分开：脚本只负责稳定、可验证地取数；Hermes Agent 负责把数据整理成适合人看的文字。

## 数据采集层

采集层的重点是时间窗口、链接、reaction 和评论上下文。

### 时间窗口

日报不是按自然日零点切，而是按固定 cutoff 计算。例如 cutoff 为 06:00：

```python
from datetime import datetime, time, timedelta
from zoneinfo import ZoneInfo


def daily_window(cutoff_hour: int = 6):
    tz = ZoneInfo("Asia/Shanghai")
    now = datetime.now(tz)
    end = datetime.combine(now.date(), time(cutoff_hour, 0), tzinfo=tz)
    if now < end:
        end -= timedelta(days=1)
    start = end - timedelta(days=1)
    return start, end
```

这样早晨 09:00 运行时，统计的是“昨天 06:00 到今天 06:00”的完整窗口，比较适合日报阅读。

### 频道消息

脚本从频道里倒序读取消息，遇到早于窗口开始时间的消息就停止。每条消息保留这些字段：

- 消息 ID；
- 发布时间；
- 正文摘要；
- reaction 列表；
- 评论数；
- 原文链接。

链接可以直接由频道名和消息 ID 拼出来：

```python
def post_link(channel: str, post_id: int) -> str:
    name = channel.removeprefix("https://t.me/").removeprefix("@")
    return f"https://t.me/{name}/{post_id}"
```

## 评论区处理

频道热度只能说明“哪条主贴被看见了”。评论区更能说明读者怎么理解这件事，所以日报里单独抓评论。

对每条有评论的频道消息，脚本继续读取关联讨论：

```python
async for comment in client.iter_messages(channel_entity, reply_to=post.id):
    # 提取评论文本、reaction、媒体信息和链接
    ...
```

评论链接同样可以构造：

```python
def comment_link(channel: str, post_id: int, comment_id: int) -> str:
    return f"{post_link(channel, post_id)}?comment={comment_id}"
```

### 噪声过滤

评论区会有很多只表达情绪、不提供信息的内容，例如单个表情、`+1`、`mark`、`蹲`、`插眼`。这些内容可以保留在数量统计里，但不适合进入观点摘要。

过滤规则不需要复杂，先抓住明显噪声即可：

```python
import re

NOISE_RE = re.compile(
    r"^(?:[\W_\s]|哈+|哈哈+|笑死|支持|顶|mark|马克|蹲|插眼|谢谢|nice|ok|1|\+1)+$",
    re.IGNORECASE,
)


def is_noise_comment(text: str) -> bool:
    compact = re.sub(r"\s+", "", text or "")
    if not compact:
        return True
    if len(compact) <= 2:
        return True
    return bool(NOISE_RE.match(compact))
```

这个过滤只决定“是否用于观点总结”，不等于删除评论。真正有 reaction 或媒体信息的评论，仍然可以进入热评候选。

## Reaction 排名

日报里保留两个榜单：

- 频道消息 Reaction Top：按主贴 reaction 总数和评论数排序。
- 评论区 Reaction Top：按评论 reaction 总数排序。

评论热评需要设置一个最低阈值，例如 reaction 总数至少为 5。原因很简单：单个 reaction 太稀疏，不能代表“多人关注”。它可以作为原始样本存在，但不适合进入热评榜。

```python
hot_posts = sorted(
    posts,
    key=lambda item: (item.reaction_total, item.replies_count),
    reverse=True,
)[:10]

hot_comments = sorted(
    [c for c in comments if c.reaction_total >= 5 and not c.noise],
    key=lambda item: item.reaction_total,
    reverse=True,
)[:10]
```

reaction 的显示也要转成人能看懂的形式。例如普通 emoji 直接显示，付费 reaction 可以显示成 `⭐`，自定义 emoji 可以显示成“自定义表情”。

## 媒体评论怎么处理

评论区里经常会出现贴纸、截图、表情包或图片。它们不一定能提供明确观点，但可能代表情绪，也可能是对主贴的补充证据。

处理原则：

- 记录媒体类型、文件名、emoji 和 reaction；
- 只有高 reaction 或确实影响讨论的媒体才下载；
- 解读图片时必须结合关联主贴和周围评论；
- 看不懂就说明它是上下文媒体，不要硬编细节。

这样可以避免把一张孤立的图误读成观点，又不会错过真正重要的图片评论。

## 输出给 Hermes Agent

采集脚本最终输出 Markdown 或 JSON。推荐先输出一份结构清楚的原始日报，再由 Hermes Agent 改写成读者友好的摘要。

原始数据可以包含：

- 频道名；
- 统计窗口；
- 消息数、评论数、有效评论数、媒体评论数；
- 频道消息 Reaction Top；
- 评论区 Reaction Top；
- 每条主贴下面的有意义评论样本。

示例结构：

```markdown
# Telegram 频道原始日报数据：<频道名>

统计窗口：<开始时间> - <结束时间>
频道消息：17；已抓评论：1194；有效评论：1020；媒体评论：123

## 频道消息 Reaction Top
1. 主题摘要……
   reaction：😁 525 🤯 93
   链接：https://t.me/<频道名>/<消息ID>

## 评论区 Reaction Top
1. 评论摘要……
   reaction：👍 42
   链接：https://t.me/<频道名>/<消息ID>?comment=<评论ID>
```

Hermes Agent 拿到这份数据后，再生成最终日报：前面是自然语言总结，后面保留榜单和原文链接。这样读者可以先快速了解重点，也可以继续点回 Telegram 看原文。

## 定时运行

这类任务适合放进 Hermes cron：

1. cron 到点运行采集脚本；
2. 脚本把原始日报输出给 Hermes Agent；
3. Hermes Agent 根据固定提示生成中文日报；
4. 最终内容发送回 Telegram。

提示词里需要说清楚三件事：

- 先写摘要，再放原始榜单；
- 评论热评只采用达到阈值的评论；
- 媒体评论要结合上下文，不要孤立猜测。

## 运行建议

- 采集脚本和摘要生成要分层，方便单独排查问题。
- 首次运行先手动测试一个短窗口，确认频道消息、评论和链接都能取到。
- 评论数很大时要设置每条主贴的评论读取上限，避免一次任务跑太久。
- 日报里保留原文链接，比只给模型总结更可靠。
- 日志保留统计数量和结果摘要即可，方便追踪任务是否正常。