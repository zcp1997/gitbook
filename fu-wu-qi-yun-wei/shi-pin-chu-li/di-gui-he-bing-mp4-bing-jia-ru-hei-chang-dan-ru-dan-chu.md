---
description: 递归查找章节目录中的 MP4，按自然排序统一转码后合并，并在每段视频之间加入黑场淡入淡出效果。
icon: film
---

# 递归合并 MP4 并加入黑场淡入淡出

## 使用场景

有一批按章节目录保存的视频，例如：

```text
/Users/zcp/Downloads/6.1-6.10/6.1-6.10
```

目录下包含 `6.1`、`6.2`、`6.3` 到 `6.10` 等子目录，每个子目录里可能还有多层目录和多个 MP4 文件。需要把所有 MP4 按章节顺序合并成一个完整视频，并在每段视频之间加入短暂黑场淡出、淡入，避免直接硬切。

这个脚本会递归查找当前目录下所有 MP4，并按照自然排序处理：

```text
6.1 → 6.2 → 6.3 → ... → 6.9 → 6.10
```

同一个章节内部如果有多个 MP4，也会按照文件路径自然排序。

## 脚本行为

- 递归查找当前目录下所有 `.mp4` 文件。
- 排除输出文件和上次残留的 `.merge_work.*` 临时目录。
- 用第一个视频的分辨率作为最终输出分辨率。
- 把所有视频统一为：
  - H.264 视频编码
  - AAC 音频编码
  - 30 FPS
  - `yuv420p`
  - 48000 Hz 双声道音频
- 每段视频开头、结尾加入约 `0.35` 秒淡入淡出。
- 没有音轨的视频会自动补静音轨，避免最终拼接失败。
- 最终生成：

```text
6.1-6.10-合并版.mp4
```

{% hint style="info" %}
FFmpeg 的 concat demuxer 要求待拼接文件具有相同的流结构、编码和时间基准。这里先逐段标准化转码，再用 concat 快速拼接，比直接把不同参数的 MP4 原样合并更稳。
{% endhint %}

## 前置依赖

脚本依赖 `ffmpeg`、`ffprobe` 和 `python3`。

macOS 可以用 Homebrew 安装 FFmpeg：

```bash
brew install ffmpeg
```

检查命令是否可用：

```bash
ffmpeg -version
ffprobe -version
python3 --version
```

## 创建脚本

进入要处理的视频根目录：

```bash
cd /Users/zcp/Downloads/6.1-6.10/6.1-6.10
```

创建 `merge_mp4.sh`：

{% code title="merge_mp4.sh" %}
```bash
#!/usr/bin/env bash
set -e

ROOT="$(pwd)"
OUTPUT="$ROOT/6.1-6.10-合并版.mp4"

# 每段视频开头、结尾的淡入淡出时长，单位：秒
FADE="0.35"

command -v ffmpeg >/dev/null 2>&1 || {
    echo "错误：没有找到 ffmpeg，请先执行：brew install ffmpeg"
    exit 1
}

command -v ffprobe >/dev/null 2>&1 || {
    echo "错误：没有找到 ffprobe，请先安装 ffmpeg"
    exit 1
}

command -v python3 >/dev/null 2>&1 || {
    echo "错误：没有找到 python3"
    exit 1
}

files=()

# 使用 Python 实现自然排序，避免 macOS 自带 sort 无法正确处理 6.10
while IFS= read -r file; do
    files+=("$file")
done < <(
    python3 - "$ROOT" "$OUTPUT" <<'PY'
import os
import re
import sys

root = os.path.abspath(sys.argv[1])
output = os.path.abspath(sys.argv[2])

def natural_key(value):
    parts = re.split(r'(\d+)', value.lower())
    return [int(part) if part.isdigit() else part for part in parts]

files = []

for current_root, dirs, names in os.walk(root):
    # 排除上一次执行意外残留的临时目录
    dirs[:] = [
        d for d in dirs
        if not d.startswith(".merge_work.")
    ]

    for name in names:
        if not name.lower().endswith(".mp4"):
            continue

        path = os.path.abspath(os.path.join(current_root, name))

        if path == output:
            continue

        files.append(path)

files.sort(key=lambda path: natural_key(os.path.relpath(path, root)))

for path in files:
    print(path)
PY
)

if [ "${#files[@]}" -eq 0 ]; then
    echo "错误：当前目录下没有找到任何 MP4 文件"
    exit 1
fi

echo "========================================"
echo "共找到 ${#files[@]} 个 MP4 文件"
echo "合并顺序如下："
echo "========================================"

index=1
for file in "${files[@]}"; do
    printf "%03d. %s\n" "$index" "${file#$ROOT/}"
    index=$((index + 1))
done

echo "========================================"

WORKDIR="$(mktemp -d "$ROOT/.merge_work.XXXXXX")"
LISTFILE="$WORKDIR/list.txt"

cleanup() {
    rm -rf "$WORKDIR"
}
trap cleanup EXIT

# 使用第一个视频的分辨率作为最终输出分辨率
IFS=, read -r WIDTH HEIGHT <<< "$(
    ffprobe \
        -v error \
        -select_streams v:0 \
        -show_entries stream=width,height \
        -of csv=p=0 \
        "${files[0]}"
)"

if [ -z "$WIDTH" ] || [ -z "$HEIGHT" ]; then
    echo "错误：无法读取第一个视频的分辨率"
    exit 1
fi

# H.264 编码通常要求宽高为偶数
WIDTH=$((WIDTH / 2 * 2))
HEIGHT=$((HEIGHT / 2 * 2))

echo "输出分辨率：${WIDTH}x${HEIGHT}"
echo "开始处理视频..."

index=1

for src in "${files[@]}"; do
    part="$(printf "%s/part_%04d.mp4" "$WORKDIR" "$index")"

    duration="$(
        ffprobe \
            -v error \
            -show_entries format=duration \
            -of default=noprint_wrappers=1:nokey=1 \
            "$src"
    )"

    if [ -z "$duration" ]; then
        echo "错误：无法读取视频时长：$src"
        exit 1
    fi

    # 防止视频本身非常短，淡入淡出时间超过视频长度
    read -r current_fade fade_out_start <<< "$(
        awk -v duration="$duration" -v fade="$FADE" '
            BEGIN {
                actual_fade = fade
                if (duration / 3 < actual_fade) {
                    actual_fade = duration / 3
                }

                printf "%.3f %.3f", actual_fade, duration - actual_fade
            }
        '
    )"

    has_audio="$(
        ffprobe \
            -v error \
            -select_streams a:0 \
            -show_entries stream=index \
            -of csv=p=0 \
            "$src"
    )"

    echo
    printf "[%d/%d] 正在处理：%s\n" "$index" "${#files[@]}" "${src#$ROOT/}"

    video_filter="scale=${WIDTH}:${HEIGHT}:force_original_aspect_ratio=decrease,pad=${WIDTH}:${HEIGHT}:(ow-iw)/2:(oh-ih)/2:black,setsar=1,fps=30,format=yuv420p,setpts=PTS-STARTPTS,fade=t=in:st=0:d=${current_fade},fade=t=out:st=${fade_out_start}:d=${current_fade}"

    audio_filter="aresample=48000,asetpts=PTS-STARTPTS,apad,atrim=duration=${duration},afade=t=in:st=0:d=${current_fade},afade=t=out:st=${fade_out_start}:d=${current_fade}"

    if [ -n "$has_audio" ]; then
        ffmpeg \
            -hide_banner \
            -loglevel warning \
            -y \
            -i "$src" \
            -map 0:v:0 \
            -map 0:a:0 \
            -vf "$video_filter" \
            -af "$audio_filter" \
            -t "$duration" \
            -c:v libx264 \
            -preset medium \
            -crf 20 \
            -video_track_timescale 90000 \
            -c:a aac \
            -b:a 192k \
            -ar 48000 \
            -ac 2 \
            -movflags +faststart \
            "$part"
    else
        # 某个视频没有音轨时，自动补充静音轨道，避免最终拼接失败
        ffmpeg \
            -hide_banner \
            -loglevel warning \
            -y \
            -i "$src" \
            -f lavfi \
            -i "anullsrc=channel_layout=stereo:sample_rate=48000" \
            -map 0:v:0 \
            -map 1:a:0 \
            -vf "$video_filter" \
            -af "$audio_filter" \
            -t "$duration" \
            -c:v libx264 \
            -preset medium \
            -crf 20 \
            -video_track_timescale 90000 \
            -c:a aac \
            -b:a 192k \
            -ar 48000 \
            -ac 2 \
            -movflags +faststart \
            "$part"
    fi

    printf "file '%s'\n" "$part" >> "$LISTFILE"

    index=$((index + 1))
done

echo
echo "开始拼接所有片段..."

ffmpeg \
    -hide_banner \
    -loglevel warning \
    -y \
    -f concat \
    -safe 0 \
    -i "$LISTFILE" \
    -c copy \
    -movflags +faststart \
    "$OUTPUT"

echo
echo "========================================"
echo "合并完成："
echo "$OUTPUT"
echo "========================================"
```
{% endcode %}

给脚本加执行权限：

```bash
chmod +x merge_mp4.sh
```

## 执行脚本

```bash
./merge_mp4.sh
```

正式转码前，脚本会先打印实际排序结果，例如：

```text
001. 6.1/某个目录/01.mp4
002. 6.2/某个目录/02.mp4
...
010. 6.10/某个目录/10.mp4
```

确认排序符合预期后等待转码和拼接完成即可。

## 调整过渡时长

当前过渡效果是稳定且适合批处理的“淡出至黑色，再从黑色淡入”。如果需要缩短或延长过渡时间，修改脚本开头这一行：

```bash
FADE="0.35"
```

例如改成半秒：

```bash
FADE="0.5"
```

## 验证结果

确认输出文件存在：

```bash
ls -lh "6.1-6.10-合并版.mp4"
```

查看最终视频参数：

```bash
ffprobe -hide_banner "6.1-6.10-合并版.mp4"
```

建议再打开视频抽查：

- 开头是否正常淡入。
- 章节之间是否有短暂黑场过渡。
- 音频是否同步淡出、淡入。
- 结尾是否完整。

## 常见问题

### 为什么不直接 concat 原始 MP4？

不同来源的视频可能分辨率、帧率、时间基准、音频采样率、音轨数量不同。直接 concat 很容易出现音画不同步、拼接失败或播放器兼容性问题。脚本先统一转码，再拼接，可以牺牲一些处理时间换稳定性。

### 为什么输出分辨率用第一个视频？

这样能避免输出尺寸频繁变化。后续视频会按比例缩放并用黑边补齐，不会强行拉伸变形。

### 某些视频没有声音怎么办？

脚本会为没有音轨的视频自动生成静音轨道，保证所有片段都有相同的音视频流结构。

## 回滚和清理

这个脚本不会修改原始 MP4。它只会在当前目录生成：

- `merge_mp4.sh`
- `6.1-6.10-合并版.mp4`

如果不再需要，删除这两个文件即可：

```bash
rm -f merge_mp4.sh "6.1-6.10-合并版.mp4"
```

临时目录 `.merge_work.*` 正常会在脚本退出时自动删除。如果脚本被强制中断后有残留，可以手动清理：

```bash
rm -rf .merge_work.*
```
