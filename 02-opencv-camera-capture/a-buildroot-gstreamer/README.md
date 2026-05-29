
# Route A：Buildroot + GStreamer 摄像头采集

本项目是 `02-camera-capture-basics` 的 Route A，目标是在当前泰山派 RK3566 的 Buildroot 系统中，不依赖 Python、不依赖 OpenCV，直接使用 GStreamer 完成摄像头采集、单帧抓图、H.264 硬件录像、文件检查和本地播放验证。

当前系统是 Buildroot，默认没有 `python3`，但已经具备 V4L2、GStreamer、RGA、MPP、KMS 等多媒体能力，因此本路线优先学习 RK3566 的原生硬件链路。

## 当前环境

开发板：泰山派 RK3566  
系统：Buildroot  
摄像头：OV5695 MIPI CSI  
主视频节点：`/dev/video0`  
图片格式：JPEG  
视频格式：H.264 裸流  
采集格式：NV12  
采集分辨率：1920x1080  
采集帧率：30fps  

已验证能力：

- `/dev/video0` 可以采集摄像头画面
- `mppjpegenc` 可以进行 JPEG 编码
- `mpph264enc` 可以进行 H.264 硬件编码
- `h264parse` 可以解析 H.264 裸流
- `mppvideodec` 可以进行 MPP 硬件解码
- `kmssink` 可以输出到本地显示设备

当前阶段先使用命令行手动验证，后续再把命令整理成脚本。

## 1. 摄像头单帧抓图

使用 GStreamer 从 `/dev/video0` 采集 1 帧 NV12 图像，并通过 `mppjpegenc` 编码为 JPEG 图片。

```bash
gst-launch-1.0 -e v4l2src device=/dev/video0 num-buffers=1 ! 'video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1' ! mppjpegenc ! filesink location=/tmp/frame.jpg
```

命令说明：

```text
gst-launch-1.0       GStreamer 命令行工具
-e                   让管线结束时正确写入文件尾部信息
v4l2src              使用 V4L2 采集设备
device=/dev/video0   指定摄像头节点
num-buffers=1        只采集 1 帧
video/x-raw          指定原始视频格式
format=NV12          指定输入格式为 NV12
width=1920           图像宽度
height=1080          图像高度
framerate=30/1       采集帧率为 30fps
mppjpegenc           使用 Rockchip MPP JPEG 编码器
filesink             写入文件
location=/tmp/frame.jpg  输出图片路径
```

## 2. 查看图片文件

抓图完成后，查看图片是否生成：

```bash
ls -lh /tmp/frame.jpg
```

如果系统支持 `file` 命令，也可以查看文件类型：

```bash
file /tmp/frame.jpg
```

如果系统支持 `du`，也可以查看文件大小：

```bash
du -h /tmp/frame.jpg
```

成功结果应该类似：

```text
-rw-r--r-- 1 root root xxxK May 29 xx:xx /tmp/frame.jpg
```

只要文件存在，并且大小不是 0，就说明 JPEG 抓图成功。

## 3. 用 multifilesrc loop=true 循环播放这张图片
因为你的系统没有 imagefreeze，可以尝试用 multifilesrc 循环读取同一张 JPEG，让它不断解码输出，相当于把单张图变成持续画面

```bash
gst-launch-1.0 multifilesrc location=/tmp/frame.jpg loop=true ! image/jpeg,framerate=1/1 ! jpegparse ! mppjpegdec ! kmssink
```
因为会持续显示图片，所以需要手动按 `Ctrl+C` 停止。


## 4. H.264 硬件录像

使用 GStreamer 从 `/dev/video0` 采集 300 帧 NV12 图像，并通过 `mpph264enc` 编码为 H.264 裸流文件。

```bash
gst-launch-1.0 -e v4l2src device=/dev/video0 num-buffers=300 ! 'video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1' ! mpph264enc ! h264parse ! filesink location=/tmp/test.h264
```

命令说明：

```text
num-buffers=300        采集 300 帧
framerate=30/1         30fps
理论录像时长             300 / 30 = 10 秒
mpph264enc             使用 Rockchip MPP H.264 硬件编码器
h264parse              解析并整理 H.264 数据流
filesink               写入文件
location=/tmp/test.h264 输出 H.264 裸流文件
```

正常输出中会看到类似：

```text
Setting pipeline to PLAYING ...
New clock: GstSystemClock
rga_api version 1.3.1_[11]
Redistribute latency...
Got EOS from element "pipeline0".
EOS received - stopping pipeline...
Execution ended after 0:00:10.x
Setting pipeline to NULL ...
Freeing pipeline ...
```

其中：

```text
Got EOS
```

表示 300 帧采集完成，管线正常结束。

如果耗时大约 10 秒，说明 1920x1080@30fps 采集和 H.264 编码基本正常。

## 5. 查看视频文件

录像完成后，查看 H.264 文件是否生成：

```bash
ls -lh /tmp/test.h264
```

如果系统支持 `file` 命令，也可以查看文件类型：

```bash
file /tmp/test.h264
```

如果系统支持 `du`，也可以查看文件大小：

```bash
du -h /tmp/test.h264
```

成功结果示例：

```text
-rw-r--r-- 1 root root 28M May 29 xx:xx /tmp/test.h264
```

只要文件存在，并且大小不是 0，就说明录像文件生成成功。

## 6. 播放 H.264 视频

当前生成的是 H.264 裸流文件，不是 MP4 文件，所以播放时需要使用 `h264parse` 解析。

使用 MPP 硬件解码并输出到本地屏幕：

```bash
gst-launch-1.0 filesrc location=/tmp/test.h264 ! h264parse ! mppvideodec ! kmssink
```

命令说明：

```text
filesrc              从文件读取 H.264 数据
h264parse            解析 H.264 裸流
mppvideodec          使用 Rockchip MPP 硬件视频解码器
kmssink              输出到本地显示设备
```

如果播放画面有问题，可以加上详细输出：

```bash
gst-launch-1.0 -v filesrc location=/tmp/test.h264 ! h264parse ! mppvideodec ! kmssink sync=false
```

如果没有接显示器，只想验证视频能不能被正常解码，可以使用：

```bash
gst-launch-1.0 filesrc location=/tmp/test.h264 ! h264parse ! mppvideodec ! fakesink
```

如果这条命令能正常跑完，说明 H.264 文件和 MPP 解码链路正常。

## 7. 本地显示说明

`kmssink` 是 KMS/DRM 显示输出，它会直接把画面显示到 RK3566 本地显示设备上。

因此：

```text
SSH / MobaXterm 窗口里不会显示图片或视频画面
```

如果想看到画面，需要连接：

- HDMI 显示器
- MIPI 屏幕
- eDP 屏幕

如果没有本地显示设备，可以使用 `fakesink` 验证解码链路是否正常。
