### **项目 1：板端视觉环境体检项目**

这个项目的目标是建立一份自己板子的“视觉能力报告”。整理成一个固定脚本，后续每次换系统、换摄像头、换模型都先跑它。

要检查：

```sh
ls /dev/video*
ls /dev/media*
ls /dev/v4l-subdev*
ls -l /dev | grep -E "rga|mpp|vpu|dri|dma|mali|video|media"
gst-inspect-1.0 | grep -i mpp
gst-inspect-1.0 | grep -i kmssink
gst-inspect-1.0 | grep -i v4l2
v4l2-ctl -d /dev/video0 --list-formats-ext
media-ctl -p
dmesg | grep -i rknpu
ls /sys/class/devfreq/
```


### 我的RK3566日志解析

```text
摄像头型号：OV5695
主视频节点：/dev/video0
最大分辨率：2592x1944
推荐采集格式：NV12
显示方式：kmssink
RGA：可用
MPP H264编码：可用
NPU驱动：rknpu 0.7.2
```

---

#### **/dev/video0 ~ /dev/video8 各节点说明**

基于 `media-ctl -p` 输出的设备拓扑，每个 video 节点的角色如下：

| 设备节点 | 实体名称 | 角色说明 |
|---|---|---|
| `/dev/video0` | `rkisp_mainpath` | **ISP 主路输出**，经过 ISP 处理后的最终图像从这里输出，是相机采集的首选节点 |
| `/dev/video1` | `rkisp_selfpath` | **ISP 辅路输出**，可输出与主路不同分辨率的图像（如主路出大图、辅路出预览小图），支持缩放 |
| `/dev/video2` | `rkisp_rawwr0` | **Raw 数据写入通道 0**，将 sensor 原始 Bayer 数据直接写入内存，不做 ISP 处理 |
| `/dev/video3` | `rkisp_rawwr2` | **Raw 数据写入通道 2**，同上，可用于多路 Raw 同时采集 |
| `/dev/video4` | `rkisp_rawwr3` | **Raw 数据写入通道 3**，同上 |
| `/dev/video5` | `rkisp_rawrd0_m` | **Raw 数据读取通道 0（主）**，将之前写入内存的 Raw 数据读回送入 ISP 重新处理 |
| `/dev/video6` | `rkisp_rawrd2_s` | **Raw 数据读取通道 2（辅）**，同上，用于 Raw 回灌处理 |
| `/dev/video7` | `rkisp-statistics` | **ISP 统计信息输出**，输出 3A（自动曝光/自动白平衡/自动对焦）统计数据，供算法库使用 |
| `/dev/video8` | `rkisp-input-params` | **ISP 参数输入**，向 ISP 写入 3A 参数或自定义处理参数，用于控制 ISP 处理行为 |

---

#### **实际采集时怎么选**

- **普通预览/拍照/录像** —— 直接用 `/dev/video0`（主路），格式选 NV12，这是最常用也是效率最高的 YUV 420 格式，MPP 硬件编码器可以直接吃进去做 H.264 编码。
- **同时出两路不同分辨率的图** —— `/dev/video0` + `/dev/video1`，比如主路出 1080p 录像、辅路出 VGA 预览。
- **需要 Raw 数据做 AI 处理** —— 用 `/dev/video2/3/4` 抓取 Bayer Raw，送到 NPU 做自定义图像处理。
- **调试 ISP 效果** —— 从 `/dev/video7` 读统计数据，通过 `/dev/video8` 写入调参参数，实现闭环调优。
