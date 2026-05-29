# 02-camera-capture-basics

本项目用于学习 RK3566 摄像头采集基础。

当前 RK3566 板端系统为 Buildroot，默认没有 `python3`，因此项目 2 拆成两条路线：

## Route A：Buildroot + GStreamer

使用当前系统已经具备的 GStreamer、V4L2、RGA、MPP 能力完成摄像头预览、抓图和录像。

目标：

- 使用 `/dev/video0` 采集 OV5695 摄像头画面
- 使用 `kmssink` 本地显示
- 使用 `mppjpegenc` 或 `jpegenc` 抓图
- 使用 `mpph264enc` 硬件编码录像
- 记录采集结果和硬件加速状态

## Route B：Buildroot + C/C++ OpenCV

检查当前 Buildroot 是否具备 C/C++ OpenCV 开发条件。如果具备，则使用 OpenCV C++ 读取 `/dev/video0` 并保存图片；如果不具备，则记录缺失项，后续通过交叉编译或重编 Buildroot 支持。

目标：

- 检查 `gcc/g++/cmake/pkg-config/opencv`
- 使用 C++ OpenCV 读取摄像头
- 保存一张图片
- 统计 FPS
- 为后续工业视觉 C++ 工程化做准备

## 当前状态

- 摄像头：OV5695 MIPI CSI
- 主视频节点：`/dev/video0`
- 当前系统：Buildroot
- Python3：未安装
- GStreamer：可用
- RGA：可用
- MPP：可用
- NPU：驱动已初始化
