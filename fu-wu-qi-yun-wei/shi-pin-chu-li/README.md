---
description: 视频转码、合并、压缩、抽帧和批处理脚本记录。
icon: video
---

# 视频处理

这里记录日常运维和内容处理里可复用的视频处理脚本，重点是把一次性命令沉淀成可以复跑、可验证、能处理异常输入的操作记录。

## 文章索引

- [XHSUHD Docker 与 ffmpeg 录播转存](xhsuhd-docker-he-ffmpeg-lu-bo-zhuan-cun.md)：使用 xhsuhd Docker 生成小红书赛事直播/回放地址，并用 ffmpeg 原样转存 4K 高帧率 MP4。
- [M3U Go 代理部署](m3u-go-dai-li-bu-shu.md)：部署 Go 写的 M3U/HLS 中转代理，改写播放列表并通过 token 鉴权保护代理链路。
- [递归合并 MP4 并加入黑场淡入淡出](di-gui-he-bing-mp4-bing-jia-ru-hei-chang-dan-ru-dan-chu.md)：递归查找章节目录中的 MP4，按自然排序统一转码后合并，并在每段视频首尾加入淡入淡出效果。
