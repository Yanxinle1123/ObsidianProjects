# FFmpeg 用法详解

> 一套命令行的音视频处理工具集，核心是 `ffmpeg`（转码/处理）、`ffprobe`（探测信息）、`ffplay`（播放）。
> 适用场景：格式转换、压缩、裁剪、合并、加水印、提取音频、截图、做 GIF、推流直播等。
> 本文侧重**实用命令 + 参数含义**，不堆砌所有选项。

---

## 1. 安装

| 平台 | 命令 |
|------|------|
| macOS (Homebrew) | `brew install ffmpeg` |
| Ubuntu/Debian | `sudo apt install ffmpeg` |
| Windows (Chocolatey) | `choco install ffmpeg` |
| Windows (Scoop) | `scoop install ffmpeg` |

> 注意：Homebrew 默认装的 `ffmpeg` 可能缺少某些非自由编码器（如 `libx264`、`libx265`）。若需要完整功能，macOS 可用 `brew install ffmpeg --with-...`（新版 Homebrew 改为通过 `ffmpeg` 包的 option 启用，或用 `brew install ffmpeg --build-from-source`）。

验证安装：

```bash
ffmpeg -version
ffprobe -version
```

---

## 2. 三个核心概念

处理音视频前必须理解三个词，否则命令很容易写错：

- **容器（Container）**：文件后缀，如 `.mp4`、`.mkv`、`.mov`。只是"盒子"，里面可以装多种编码的流。
- **编码（Codec）**：真正压缩数据的算法。视频常见 `H.264 (libx264)`、`H.265 (libx265)`、`VP9`、`AV1`；音频常见 `AAC`、`MP3`、`Opus`。
- **流（Stream）**：一个媒体文件通常含多条流——1 路视频、1~n 路音频、可选字幕。用 `-map` 选择。

```bash
# 查看文件信息（编码、分辨率、时长、流数量）
ffprobe -v error -show_entries format=duration,size,bit_rate -of default=noprint_wrappers=1 input.mp4
ffprobe -v error -show_streams input.mp4
```

---

## 3. 最常用命令速查

| 需求 | 命令 |
|------|------|
| MP4 → MKV | `ffmpeg -i in.mp4 out.mkv` |
| 提取音频(AAC) | `ffmpeg -i in.mp4 -vn -acodec copy out.aac` |
| 转成 MP3 | `ffmpeg -i in.mp4 -vn -ab 192k out.mp3` |
| 仅提取视频 | `ffmpeg -i in.mp4 -an -vcodec copy out.mp4` |
| 截图一张 | `ffmpeg -i in.mp4 -ss 00:00:05 -vframes 1 frame.jpg` |
| 截成 GIF | `ffmpeg -i in.mp4 -ss 2 -t 3 -vf "fps=10,scale=480:-1" out.gif` |
| 压缩体积 | `ffmpeg -i in.mp4 -crf 28 out.mp4` |
| 改变分辨率 | `ffmpeg -i in.mp4 -vf scale=1280:720 out.mp4` |
| 裁剪片段 | `ffmpeg -i in.mp4 -ss 00:01:00 -to 00:02:00 -c copy cut.mp4` |
| 合并多个文件 | `ffmpeg -f concat -i list.txt -c copy merged.mp4` |

> `-c copy` / `-codec copy` 表示不重新编码，速度极快、无损，但只能用于同容器兼容的流；裁剪若不用 `-c copy` 则重新编码更精确。

---

## 4. 关键参数详解

### 通用参数

| 参数 | 含义 |
|------|------|
| `-i` | 输入文件（可多个） |
| `-y` | 覆盖输出文件不询问 |
| `-n` | 输出文件存在则退出不覆盖 |
| `-ss` | 起始时间（放在 `-i` 前更快，但部分格式不精确） |
| `-t` | 持续时间（秒或 `hh:mm:ss`） |
| `-to` | 结束时间 |
| `-r` | 帧率 |
| `-threads` | 线程数（如 `-threads 4`） |

### 视频编码参数（libx264）

| 参数 | 作用 |
|------|------|
| `-c:v libx264` | 使用 H.264 编码 |
| `-crf` | 质量系数，**18~28** 常用，越小越清晰体积越大（默认 23） |
| `-preset` | 编码速度/压缩率权衡：`ultrafast` → `veryslow`，越慢压缩越好 |
| `-b:v` | 固定码率（如 `-b:v 2M`） |
| `-maxrate` / `-bufsize` | 限制最大码率（直播/平台上传常用） |
| `-pix_fmt yuv420p` | 兼容性像素格式（微信/网页播放必备） |

### 音频编码参数

| 参数 | 作用 |
|------|------|
| `-c:a aac` | AAC 编码 |
| `-b:a` | 音频码率（如 `-b:a 128k`） |
| `-ar` | 采样率（如 `-ar 44100`） |
| `-ac` | 声道数（`-ac 2` 立体声，`-ac 1` 单声道） |

---

## 5. 高频实战场景

### 5.1 压缩视频（最常用）

```bash
# 快速压缩：CRF 28，预设 medium
ffmpeg -i input.mp4 -c:v libx264 -crf 28 -preset medium -c:a aac -b:a 128k output.mp4
```

### 5.2 精确裁剪（重新编码，时间准）

```bash
ffmpeg -i in.mp4 -ss 00:00:10 -to 00:00:25 -c:v libx264 -c:a aac clip.mp4
```

### 5.3 合并视频（同规格）

先建 `list.txt`：

```text
file 'part1.mp4'
file 'part2.mp4'
file 'part3.mp4'
```

```bash
ffmpeg -f concat -safe 0 -i list.txt -c copy merged.mp4
```

> `-safe 0` 允许使用绝对路径。若合并失败多为编码不一致，需先统一转码再合并。

### 5.4 加文字水印

```bash
ffmpeg -i in.mp4 -vf "drawtext=text='Claw':fontcolor=white:fontsize=48:x=20:y=20" out.mp4
```

### 5.5 加图片水印

```bash
ffmpeg -i in.mp4 -i logo.png -filter_complex "overlay=W-w-20:20" out.mp4
```

### 5.6 批量截图（每 1 秒一帧）

```bash
ffmpeg -i in.mp4 -vf fps=1 thumb_%03d.jpg
```

### 5.7 视频转 GIF（控制体积）

```bash
ffmpeg -i in.mp4 -ss 3 -t 4 -vf "fps=15,scale=640:-1:flags=lanczos,split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse" out.gif
```

### 5.8 提取某段时间音频并降噪

```bash
ffmpeg -i in.mp4 -ss 00:01:00 -to 00:02:00 -vn -c:a libmp3lame -b:a 192k clip.mp3
```

### 5.9 调整播放速度

```bash
# 视频加速 2 倍，音频同步（atempo 最多 2x，叠加实现 4x）
ffmpeg -i in.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v];[0:a]atempo=2.0[a]" -map "[v]" -map "[a]" fast.mp4
```

### 5.10 推流到 RTMP（直播）

```bash
ffmpeg -re -i in.mp4 -c:v libx264 -preset veryfast -c:a aac -f flv rtmp://server/app/stream
```

> `-re` 表示按原始帧率读取（实时推流必须加），否则会瞬间推完。

---

## 6. 滤镜（filter）进阶

滤镜用 `-vf`（视频）或 `-af`（音频）或 `-filter_complex`（多输入/多路）。

常用视频滤镜：

| 滤镜 | 作用 |
|------|------|
| `scale=w:h` | 缩放（如 `scale=1920:1080`，`-1` 表示按比例） |
| `crop=w:h:x:y` | 裁剪区域 |
| `rotate=PI/2` | 旋转 90° |
| `transpose=1` | 顺时针转 90°（手机竖屏转正常用） |
| `hflip` / `vflip` | 水平/垂直翻转 |
| `fade=in:0:30` | 前 30 帧淡入 |
| `gblur=sigma=3` | 高斯模糊 |
| `eq=brightness=0.1:contrast=1.2` | 亮度/对比度 |
| `drawtext=...` | 加文字（见 5.4） |
| `subtitles=sub.srt` | 烧录字幕 |

音频滤镜：

| 滤镜 | 作用 |
|------|------|
| `volume=0.5` | 音量降到一半 |
| `atempo=1.5` | 变速（0.5~2.0） |
| `highpass=f=200` | 高通滤波（去低频噪声） |
| `lowpass=f=3000` | 低通滤波 |
| `afftdn` | 频谱降噪 |

示例：裁剪 + 缩放 + 淡入

```bash
ffmpeg -i in.mp4 -vf "crop=800:600:0:0,scale=400:300,fade=in:0:25" out.mp4
```

---

## 7. 硬件加速（macOS / NVIDIA）

### macOS（VideoToolbox，Apple 芯片友好）

```bash
# H.264 硬件编码
ffmpeg -i in.mp4 -c:v h264_videotoolbox -b:v 4M out.mp4
# H.265 硬件编码
ffmpeg -i in.mp4 -c:v hevc_videotoolbox -b:v 4M out.mp4
```

> 你的 M1 Pro 用 VideoToolbox 比 CPU 编码快很多，但 CRF 控制不适用，改用 `-b:v` 固定码率。

### NVIDIA（仅参考，你无 N 卡）

```bash
ffmpeg -i in.mp4 -c:v h264_nvenc -preset p1 out.mp4
```

---

## 8. ffprobe 探测技巧

```bash
# 只看关键流信息
ffprobe -v error -show_entries stream=index,codec_type,codec_name,width,height,bit_rate -of default=noprint_wrappers=1 input.mp4

# 输出 JSON（脚本解析友好）
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4

# 查看关键帧间隔（ GOP ）
ffprobe -v error -select_streams v:0 -show_frames -show_entries frame=key_frame,pict_type -of csv input.mp4 | grep -c ",I,"
```

---

## 9. 常见坑

1. **裁剪用 `-c copy` 不精确**：因为切割点不一定在关键帧，结尾会多一小段。要精确就重新编码（去掉 `-c copy`）。
2. **`-ss` 位置影响速度**：放在 `-i` 前是"快速定位"（跳到关键帧），放后面是"解码到指定点"（慢但准）。
3. **微信/网页播放黑屏**：多半是像素格式问题，加 `-pix_fmt yuv420p`。
4. **合并失败**：先确认所有片段编码一致，不一致就统一转码。
5. **中文路径/文件名**：macOS 一般 OK，但脚本里建议用引号包裹路径。
6. **音画不同步**：推流/变速时检查 `-vsync` 和 `atempo` 链路。

---

## 10. 一条命令模板（可复用）

```bash
ffmpeg -y -i "$INPUT" \
  -c:v libx264 -crf 23 -preset medium -pix_fmt yuv420p \
  -c:a aac -b:a 128k -ar 44100 -ac 2 \
  -movflags +faststart \
  "$OUTPUT"
```

> `-movflags +faststart` 把 moov atom 移到文件头，网页端可边下边播，上传视频平台强烈建议加。

---

## 参考

- 官方文档：https://ffmpeg.org/documentation.html
- 滤镜文档：https://ffmpeg.org/ffmpeg-filters.html
- 常用命令速查（英文）：https://ffmpeg.org/ffmpeg.html#Options
