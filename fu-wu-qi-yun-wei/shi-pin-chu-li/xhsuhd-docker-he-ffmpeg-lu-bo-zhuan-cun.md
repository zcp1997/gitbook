---
description: 使用 xhsuhd Docker 生成小红书赛事回放地址，并用 ffmpeg 原样转存为本地 MP4。
icon: file-video
---

# XHSUHD Docker 与 ffmpeg 录播转存

## 场景

把 XHSUHD Docker 服务提供的赛事回放地址，使用 `ffmpeg` 原样转存为本地 MP4 文件。适合需要保留 4K 高帧率回放、避免播放器临时拉流失败、或需要后续离线剪辑的场景。

{% hint style="warning" %}
XHSUHD 依赖当前小红书 Web 登录态。`a1` 和 `web_session` 属于账号相关 Cookie，不要写进公开文档、聊天记录或 Git 仓库。实际部署时放到 `.env`，并确保 `.env` 不提交。
{% endhint %}

## Docker 来源与本地目录

- 镜像：`iptvtop/xhsuhd:latest`
- 容器名：`xhsuhd`
- 容器端口：`34567`
- 本机映射端口：`34567`
- 本地 Compose 目录：`/Users/zcp/docker-apps/xhsuhd`
- Compose 文件：`/Users/zcp/docker-apps/xhsuhd/docker-compose.yml`
- 环境变量文件：`/Users/zcp/docker-apps/xhsuhd/.env`

`docker-compose.yml`：

```yaml
services:
  xhsuhd:
    image: iptvtop/xhsuhd:latest
    container_name: xhsuhd
    restart: always
    ports:
      - "34567:34567"
    environment:
      XHS_A1: ${XHS_A1}
      XHS_WEB_SESSION: ${XHS_WEB_SESSION}
```

`.env` 示例：

```env
XHS_A1=从 Chrome Cookie 获取的 a1
XHS_WEB_SESSION=从 Chrome Cookie 获取的 web_session
```

启动或更新：

```bash
cd /Users/zcp/docker-apps/xhsuhd
docker compose pull && docker compose up -d --force-recreate
```

查看状态：

```bash
docker ps --filter name=xhsuhd
docker logs --tail 80 xhsuhd
```

播放列表：

```text
http://127.0.0.1:34567/xhslist.m3u
```

局域网设备访问时，把 `127.0.0.1` 换成运行 Docker 的机器 IP，例如：

```text
http://192.168.10.12:34567/xhslist.m3u
```

## ffmpeg 转存录播

本次转存目标：

- 回放名称：`世界杯4K回放-韩国vs捷克`
- 源地址：`http://192.168.10.12:34567/replay/6a2b8b570000000007020226`
- 输出文件：`/Users/zcp/Downloads/世界杯4K回放-韩国vs捷克.mp4`

先探测源：

```bash
ffprobe -hide_banner -v warning \
  -show_entries format=format_name,duration \
  -of default=noprint_wrappers=1:nokey=0 \
  'http://192.168.10.12:34567/replay/6a2b8b570000000007020226'
```

转存命令：

```bash
ffmpeg -hide_banner -y \
  -i 'http://192.168.10.12:34567/replay/6a2b8b570000000007020226' \
  -c copy \
  -movflags +faststart \
  '/Users/zcp/Downloads/世界杯4K回放-韩国vs捷克.mp4'
```

参数说明：

- `-hide_banner`：减少 ffmpeg 启动信息，日志更干净。
- `-y`：如果目标文件已存在则覆盖。覆盖前应确认目标路径无误。
- `-i`：输入 XHSUHD 回放地址。
- `-c copy`：不重新编码，直接复制视频和音频流，速度快且不损失画质。
- `-movflags +faststart`：把 MP4 的 `moov atom` 移到文件开头，便于播放器更快开始播放。

{% hint style="warning" %}
`-y` 会覆盖同名文件。重要文件建议先改输出文件名，或在执行前检查目标文件是否存在。
{% endhint %}

## 验证

确认文件存在和大小：

```bash
python3 - <<'PY'
from pathlib import Path
p = Path('/Users/zcp/Downloads/世界杯4K回放-韩国vs捷克.mp4')
print('exists=', p.exists())
if p.exists():
    size = p.stat().st_size
    print('bytes=', size)
    print('gib=', round(size / 1024 / 1024 / 1024, 3))
PY
```

确认视频参数：

```bash
ffprobe -hide_banner -v error \
  -show_entries format=duration,size,format_name:stream=index,codec_type,codec_name,width,height,avg_frame_rate \
  -of json \
  '/Users/zcp/Downloads/世界杯4K回放-韩国vs捷克.mp4'
```

本次验证结果：

- 文件大小：`6,458,541,325 bytes`，约 `6.015 GiB`
- 容器格式：MP4
- 视频编码：HEVC / H.265
- 分辨率：`3840x2160`
- 帧率：`50 fps`
- 音频编码：AAC
- 时长：约 `6383.25 秒`，即 `1小时46分23秒`

## 常见问题

### 列表或回放为空

通常是当前没有直播赛事，或 `a1` / `web_session` 失效。重新在 Chrome 登录小红书，再更新 `.env` 后重启容器。

```bash
cd /Users/zcp/docker-apps/xhsuhd
docker compose up -d --force-recreate
```

### 局域网设备打不开地址

检查运行 Docker 的机器 IP、端口映射和防火墙。服务正常时本机应能访问：

```bash
curl -I http://127.0.0.1:34567/xhslist.m3u
```

局域网设备使用：

```text
http://运行Docker机器的局域网IP:34567/xhslist.m3u
```

### ffmpeg 下载慢或中断

可以重新执行同一条命令。若担心覆盖已有完整文件，先换一个输出文件名，再用 `ffprobe` 对比时长和大小。
