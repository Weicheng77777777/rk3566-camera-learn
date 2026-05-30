# **泰山派 RK3566 Ubuntu 22.04 修复 AP6212 Wi-Fi 固件加载失败记录**

时间：2026 年 5 月 31 日  
开发板：立创泰山派 RK3566  
系统：Ubuntu 22.04  
内核：Linux 4.19.232  
Wi-Fi 模组：AP6212 / BCM43438A1  
驱动：`bcmdhd.ko`

这次修 Wi-Fi 的过程，其实不是简单地“缺一个文件，复制一下就好”。一开始我以为是驱动没加载、DTS 没配好、SDIO 没识别，后来通过串口日志一步一步排查，最终确认真正的问题是：**Ubuntu rootfs 里没有 Rockchip 驱动期望的 `/vendor/etc/firmware/` 目录，导致 `bcmdhd` 找不到 Broadcom Wi-Fi 固件。**

更麻烦的是，修好 rootfs 后，重新打包出来的 `update.img` 又因为体积太大，在 RKDevTool 里一直提示“加载固件失败”。最后我绕过整包升级，改用 **分区镜像下载**，终于成功把新的 rootfs 烧录进开发板。

这篇记录完整复盘这次 Wi-Fi 修复过程。

---

## **一、问题现象**

系统已经成功启动到 Ubuntu 22.04，eDP 屏幕也已经有显示，串口可以登录系统。

但是 Wi-Fi 不能使用。

执行：

```bash
ip link
nmcli dev
```

能看到 `wlan0`，但是状态异常：

```text
wlan0   wifi   unavailable
```

尝试手动拉起 Wi-Fi：

```bash
sudo ip link set wlan0 up
```

返回：

```text
RTNETLINK answers: Operation not permitted
```

一开始这个提示很容易误导人，以为是权限问题。但命令已经用了 `sudo`，所以问题不在用户权限，而在内核驱动初始化失败。

---

## **二、第一次抓关键日志**

为了避免 NetworkManager 一直反复拉起 Wi-Fi，先停掉相关服务：

```bash
sudo systemctl stop NetworkManager
sudo systemctl stop wpa_supplicant
sudo systemctl stop ModemManager 2>/dev/null || true
sudo systemctl stop systemd-networkd 2>/dev/null || true
```

然后清空内核日志，再手动拉起 `wlan0`：

```bash
sudo dmesg -C
sudo ip link set wlan0 down 2>/dev/null || true
sudo ip link set wlan0 up

dmesg | grep -Ei "dhd|bcmdhd|wlan|firmware|ap6212|43438|clm|nvram|sdio|mmc2|rfkill|download|dongle" | tail -n 250
```

日志里出现了最关键的一句：

```text
dhdsdio_download_code_file: Open firmware file failed /vendor/etc/firmware/fw_bcm43438a1.bin
_dhdsdio_download_firmware: dongle image file download failed
dhd_bus_devreset Failed to download binary to the dongle
[dhd-wlan0] wl_android_wifi_on : Failed
```

这说明 Wi-Fi 驱动不是完全没工作，而是已经执行到了下载固件阶段，只是找不到固件文件。

---

## **三、确认不是 SDIO、不是电源、不是 DTS 主体问题**

同一段日志里还有这些信息：

```text
[WLAN_RFKILL]: rockchip_wifi_power: 1
[WLAN_RFKILL]: wifi turn on power [GPIO-1-0]
sdio_reset_comm():
mmc2: new high speed SDIO card at address 0001
sdioh_start: set sd_f2_blocksize 256
dhd_bus_devreset: == WLAN ON ==
DHD: dongle ram size is set to 524288(orig 524288) at 0x0
```

这些信息说明：

Wi-Fi 电源控制是正常的。  
SDIO 总线是正常的。  
`mmc2` 能识别到 SDIO Wi-Fi 设备。  
`bcmdhd` 驱动已经开始和 Wi-Fi 芯片通信。  
真正失败点是固件文件无法打开。

所以排查方向从“驱动是否工作”转成了“rootfs 里是否存在驱动需要的固件路径”。

---

## **四、确认 `/vendor/etc/firmware/` 根本不存在**

在板端执行：

```bash
ls -lah /vendor
ls -lah /vendor/etc
ls -lah /vendor/etc/firmware
```

结果：

```text
ls: cannot access '/vendor': No such file or directory
ls: cannot access '/vendor/etc': No such file or directory
ls: cannot access '/vendor/etc/firmware': No such file or directory
```

这就非常明确了。

驱动期望路径是：

```text
/vendor/etc/firmware/fw_bcm43438a1.bin
/vendor/etc/firmware/nvram_ap6212a.txt
/vendor/etc/firmware/config.txt
/vendor/etc/firmware/clm_bcm43438a1.blob
```

但是 Ubuntu rootfs 里连 `/vendor` 目录都没有。

这是因为这套 Rockchip Wi-Fi 驱动原本更偏 Android / Buildroot 风格，默认从 `/vendor/etc/firmware/` 找固件。而我移植 Ubuntu 22.04 时，只做了 Linux rootfs，没有把 Rockchip 的 vendor firmware 目录带进去。

---

## **五、确认 Wi-Fi 芯片型号和固件文件**

从启动日志中可以看到：

```text
[WLAN_RFKILL]: wlan_platdata_parse_dt: wifi_chip_type = ap6212
```

AP6212 对应 Broadcom BCM43438A1。

在 SDK 里查找固件：

```bash
cd ~/tspi-linux-4.9

find external/rkwifibt/firmware/broadcom -iname "*43438*"
find external/rkwifibt/firmware/broadcom -iname "nvram_ap6212*"
find external/rkwifibt/firmware/broadcom -iname "config.txt"
```

最终用到的主要文件是：

```text
external/rkwifibt/firmware/broadcom/AP6212A1/wifi/fw_bcm43438a1.bin
external/rkwifibt/firmware/broadcom/AP6212A1/wifi/nvram_ap6212a.txt
external/rkwifibt/firmware/broadcom/AP6203BM/wifi/config.txt
```

其中最关键的是：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
```

`config.txt` 也放进去，方便驱动读取配置。

`clm_bcm43438a1.blob` 当时 SDK 里没有找到，所以先没有强行补。因为当前错误发生在主固件 `fw_bcm43438a1.bin` 打不开，还没到 CLM 阶段。

---

## **六、把 Wi-Fi 固件固化进 Ubuntu rootfs 模板**

注意，这里不能只在板子上临时创建 `/vendor/etc/firmware/`，因为重新烧录后会丢。正确做法是把固件放进宿主机上的 Ubuntu rootfs 模板里。

我的 rootfs 模板路径是：

```text
~/tspi-ubuntu22-rootfs/ubuntu22-rootfs
```

SDK 路径是：

```text
~/tspi-linux-4.9
```

执行：

```bash
cd ~/tspi-linux-4.9

sudo mkdir -p ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware

sudo cp -a external/rkwifibt/firmware/broadcom/AP6212A1/wifi/fw_bcm43438a1.bin \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/

sudo cp -a external/rkwifibt/firmware/broadcom/AP6212A1/wifi/nvram_ap6212a.txt \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/

sudo cp -a external/rkwifibt/firmware/broadcom/AP6203BM/wifi/config.txt \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/
```

然后修正权限：

```bash
sudo find ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware -type d -exec chmod 755 {} \;
sudo find ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware -type f -exec chmod 644 {} \;

sync
```

检查：

```bash
ls -lah ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware
```

看到：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
config.txt
```

说明固件已经进了 rootfs 模板。

我也顺手复制了一份到 `/lib/firmware/`，虽然当前 `bcmdhd` 明确找 `/vendor/etc/firmware/`，但放一份备份更稳：

```bash
sudo mkdir -p ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware

sudo cp -a ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/fw_bcm43438a1.bin \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/

sudo cp -a ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/nvram_ap6212a.txt \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/

sudo cp -a ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/vendor/etc/firmware/config.txt \
  ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/

sudo chmod 644 ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/fw_bcm43438a1.bin
sudo chmod 644 ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/nvram_ap6212a.txt
sudo chmod 644 ~/tspi-ubuntu22-rootfs/ubuntu22-rootfs/lib/firmware/config.txt

sync
```

---

## **七、制作新的 rootfs.img**

一开始我做了 4GB 的 rootfs：

```bash
dd if=/dev/zero of="$ROOTFS_IMG" bs=1M count=4096 status=progress
```

后来发现生成的 `update.img` 超过 4GB，RKDevTool 加载失败。之后改成 3800MB、3072MB 继续测试。最终经验是：**rootfs 不需要做太大，实际内容不到 1GB，烧录后可以在板端用 `resize2fs` 扩容。**

重新生成 rootfs 的核心流程如下：

```bash
cd ~/tspi-ubuntu22-rootfs

sudo umount -lf rootfs-mnt 2>/dev/null || true
sudo umount -lf rootfs-check 2>/dev/null || true

ROOTFS_DIR="$PWD/ubuntu22-rootfs"
ROOTFS_IMG="$PWD/rootfs.img"
MNT_DIR="$PWD/rootfs-mnt"

sudo rm -f "$ROOTFS_IMG"
sudo rm -rf "$MNT_DIR"
mkdir -p "$MNT_DIR"
```

创建 rootfs 镜像，例如 3072MB：

```bash
dd if=/dev/zero of="$ROOTFS_IMG" bs=1M count=3072 status=progress
```

格式化：

```bash
mkfs.ext4 -F -L rootfs "$ROOTFS_IMG"
```

挂载：

```bash
sudo mount -o loop "$ROOTFS_IMG" "$MNT_DIR"
```

复制 rootfs 模板：

```bash
sudo rsync -aHAX --numeric-ids "$ROOTFS_DIR"/ "$MNT_DIR"/
```

检查固件是否真的被复制进镜像：

```bash
ls -lah "$MNT_DIR/vendor/etc/firmware"
```

必须看到：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
config.txt
```

同步并卸载：

```bash
sync
sudo umount "$MNT_DIR"
```

检查镜像：

```bash
e2fsck -fy "$ROOTFS_IMG"
ls -lh "$ROOTFS_IMG"
file "$ROOTFS_IMG"
```

输出类似：

```text
Linux rev 1.0 ext4 filesystem data
```

说明 rootfs 镜像正常。

---

## **八、替换 SDK 里的 rootfs.ext4 和 rootfs.img**

这里有一个坑。

我一开始以为只要替换：

```text
rockdev/rootfs.img
```

就够了。

但后来执行：

```bash
file ~/tspi-linux-4.9/rockdev/rootfs.img
```

发现它是软链接：

```text
symbolic link to ../buildroot/output/rockchip_rk3566/images/rootfs.ext2
```

而 `mkfirmware.sh` 输出里又出现：

```text
Linking rootfs.img from /home/chen20/tspi-linux-4.9/rockdev/rootfs.ext4...
```

说明打包流程里 `rootfs.ext4` 和 `rootfs.img` 的关系比较绕。

最终我采用最稳的做法：**强制替换 `rockdev/rootfs.ext4`，然后让 `rootfs.img` 指向它。**

```bash
cd ~/tspi-linux-4.9

cp -f ~/tspi-ubuntu22-rootfs/rootfs.img rockdev/rootfs.ext4

rm -f rockdev/rootfs.img
ln -s rootfs.ext4 rockdev/rootfs.img

sync
```

检查：

```bash
ls -lhL rockdev/rootfs.ext4
ls -lhL rockdev/rootfs.img
file rockdev/rootfs.ext4
readlink -f rockdev/rootfs.img
```

目标状态：

```text
rootfs.ext4 是新的 ext4 rootfs
rootfs.img -> rootfs.ext4
rootfs.img 真实大小和 rootfs.ext4 一致
```

---

## **九、重新整理 rockdev 镜像**

执行：

```bash
cd ~/tspi-linux-4.9
./mkfirmware.sh
```

过程中出现过这个 warning：

```text
warning: recovery.img not found
```

这是之前就遇到过的问题。我的系统没有生成 `recovery.img`，所以需要在打包配置里注释掉 recovery。

检查 package-file：

```bash
grep -nE "rootfs|recovery" tools/linux/Linux_Pack_Firmware/rockdev/package-file
```

正确状态：

```text
13:#recovery Image/recovery.img
14:rootfs               Image/rootfs.img
```

如果 recovery 没有注释，执行：

```bash
sed -i 's/^\(recovery[[:space:]]\)/#\1/' tools/linux/Linux_Pack_Firmware/rockdev/package-file
```

再检查一次：

```bash
grep -nE "rootfs|recovery" tools/linux/Linux_Pack_Firmware/rockdev/package-file
```

---

## **十、重新打包 update.img，以及为什么最后放弃整包加载**

真正打包 `update.img` 的脚本不在 SDK 根目录，而是在：

```text
tools/linux/Linux_Pack_Firmware/rockdev/mkupdate.sh
```

一开始我在根目录执行：

```bash
./mkupdate.sh
```

结果报错：

```text
bash: ./mkupdate.sh: No such file or directory
```

正确命令是：

```bash
cd ~/tspi-linux-4.9/tools/linux/Linux_Pack_Firmware/rockdev
./mkupdate.sh
```

打包后检查：

```bash
stat ~/tspi-linux-4.9/rockdev/update.img
file ~/tspi-linux-4.9/rockdev/update.img
```

我的包显示：

```text
Size: 4035287044
file: data
```

为了确认 `update.img` 本身是不是坏的，我用 `afptool` 解包：

```bash
cd ~/tspi-linux-4.9/tools/linux/Linux_Pack_Firmware/rockdev

./afptool -unpack ~/tspi-linux-4.9/rockdev/update.img /tmp/test_update_unpack
```

输出：

```text
Android Firmware Package Tool v2.0
Check file... OK
------- UNPACK ------
package-file
Image/MiniLoaderAll.bin
Image/parameter.txt
Image/uboot.img
Image/misc.img
Image/boot.img
Image/rootfs.img
Image/oem.img
Image/userdata.img
Unpack firmware OK!
------ OK ------
```

这说明 `update.img` 本身是有效包。

但是在 Windows 的 RKDevTool 里仍然提示：

```text
加载固件失败
```

官方包可以加载，我自己打的包不能加载。结合文件大小接近 4GB，我判断 RKDevTool 对大体积 update 整包兼容不好。即使 `afptool` 能解包，RKDevTool 也可能因为大文件、路径、版本或内部限制加载失败。

最后我决定不再使用“升级固件”整包方式，而是改用 **下载镜像 / 分区烧录**。

---

## **十一、准备分区烧录文件**

这些文件可以直接从 `rockdev/` 目录拿：

```text
MiniLoaderAll.bin
parameter.txt
uboot.img
boot.img
rootfs.img
misc.img
oem.img
userdata.img
```

我创建一个单独目录，方便拷贝到 Windows：

```bash
cd ~/tspi-linux-4.9

mkdir -p ~/tspi-burn-images

cp -avL rockdev/MiniLoaderAll.bin ~/tspi-burn-images/
cp -avL rockdev/parameter.txt     ~/tspi-burn-images/
cp -avL rockdev/uboot.img         ~/tspi-burn-images/
cp -avL rockdev/misc.img          ~/tspi-burn-images/
cp -avL rockdev/boot.img          ~/tspi-burn-images/
cp -avL rockdev/rootfs.img        ~/tspi-burn-images/
cp -avL rockdev/oem.img           ~/tspi-burn-images/
cp -avL rockdev/userdata.img      ~/tspi-burn-images/
```

这里一定要用：

```bash
cp -avL
```

因为 `rootfs.img` 是软链接，`-L` 可以复制软链接指向的真实文件，而不是只复制一个链接。

检查：

```bash
ls -lh ~/tspi-burn-images
```

确认有：

```text
MiniLoaderAll.bin
parameter.txt
uboot.img
misc.img
boot.img
rootfs.img
oem.img
userdata.img
```

再次挂载确认 `rootfs.img` 里真的有固件：

```bash
mkdir -p /tmp/rootfs-check
sudo mount -o loop ~/tspi-burn-images/rootfs.img /tmp/rootfs-check

ls -lah /tmp/rootfs-check/vendor/etc/firmware

sudo umount /tmp/rootfs-check
```

必须看到：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
config.txt
```

---

## **十二、RKDevTool 分区烧录对应关系**

在 RKDevTool 的“下载镜像”页面里，不使用整包 `update.img`，而是逐行选择镜像。

对应关系如下：

| RKDevTool 分区名 | 文件 |
|---|---|
| `loader` | `MiniLoaderAll.bin` |
| `parameter` | `parameter.txt` |
| `uboot` | `uboot.img` |
| `misc` | `misc.img` |
| `boot` | `boot.img` |
| `rootfs` | `rootfs.img` |
| `oem` | `oem.img` |
| `userdata` | `userdata.img` |

有一行需要特别注意：

```text
recovery
```

我没有 `recovery.img`，所以这一行必须取消勾选，不要烧。

这次修 Wi-Fi 最关键的是：

```text
rootfs -> rootfs.img
```

因为 `/vendor/etc/firmware/` 是在 rootfs 里面。

---

## **十三、这次过程中出错的命令和踩坑点**

第一个错误是把说明文字当成真实文件名执行了：

```bash
cp -a 生成的rootfs.img ~/tspi-linux-4.9/rockdev/rootfs.img
```

报错：

```text
cp: cannot stat '生成的rootfs.img': No such file or directory
```

原因很简单，`生成的rootfs.img` 是说明里的占位文字，不是真实文件。真实路径应该是：

```bash
cp -f ~/tspi-ubuntu22-rootfs/rootfs.img ~/tspi-linux-4.9/rockdev/rootfs.img
```

后来进一步修正为复制到：

```bash
cp -f ~/tspi-ubuntu22-rootfs/rootfs.img ~/tspi-linux-4.9/rockdev/rootfs.ext4
```

第二个错误是在 SDK 根目录执行了不存在的脚本：

```bash
./mkupdate.sh
```

报错：

```text
bash: ./mkupdate.sh: No such file or directory
```

正确位置是：

```bash
cd ~/tspi-linux-4.9/tools/linux/Linux_Pack_Firmware/rockdev
./mkupdate.sh
```

第三个坑是 `rootfs.img` 是软链接：

```bash
file ~/tspi-linux-4.9/rockdev/rootfs.img
```

输出：

```text
symbolic link to ...
```

所以不能只看 `ls -lh`，要用：

```bash
ls -lhL rockdev/rootfs.img
readlink -f rockdev/rootfs.img
```

第四个坑是 `recovery.img` 缺失。

`mkfirmware.sh` 里会提示：

```text
warning: recovery.img not found
```

而打包时如果 `package-file` 里启用了 recovery，就会导致打包异常。最终处理方式是注释 recovery：

```bash
sed -i 's/^\(recovery[[:space:]]\)/#\1/' tools/linux/Linux_Pack_Firmware/rockdev/package-file
```

确认：

```bash
grep -nE "rootfs|recovery" tools/linux/Linux_Pack_Firmware/rockdev/package-file
```

正确状态：

```text
#recovery Image/recovery.img
rootfs Image/rootfs.img
```

第五个坑是 RKDevTool 加载大体积 `update.img` 失败。

虽然：

```bash
./afptool -unpack update.img /tmp/test_update_unpack
```

能成功解包，但 Windows RKDevTool 仍然提示：

```text
加载固件失败
```

最后绕过整包加载，改用分区烧录成功。

第六个小问题是板端没有 `rfkill` 命令：

```bash
rfkill list
```

报错：

```text
rfkill: command not found
```

这不是根因。可以通过 sysfs 简单查看：

```bash
cat /sys/class/rfkill/rfkill*/state 2>/dev/null
cat /sys/class/rfkill/rfkill*/type 2>/dev/null
cat /sys/class/rfkill/rfkill*/name 2>/dev/null
```

真正的根因依然是固件路径不存在。

---

## **十四、最终如何验证成功**

烧录成功后，第一步不是直接连 Wi-Fi，而是先确认新的 rootfs 是否真的烧进去了。

进入板端 Ubuntu 后执行：

```bash
ls -lah /vendor
ls -lah /vendor/etc
ls -lah /vendor/etc/firmware
```

必须看到：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
config.txt
```

然后停掉 NetworkManager，手动测试驱动初始化：

```bash
sudo systemctl stop NetworkManager
sudo systemctl stop wpa_supplicant

sudo dmesg -C

sudo ip link set wlan0 down 2>/dev/null || true
sudo ip link set wlan0 up
```

再看日志：

```bash
dmesg | grep -Ei "dhd|bcmdhd|wlan|firmware|ap6212|43438|clm|nvram|sdio|mmc2|download|dongle" | tail -n 250
```

这次最重要的是不应该再出现：

```text
Open firmware file failed /vendor/etc/firmware/fw_bcm43438a1.bin
```

如果这条错误消失，说明 Wi-Fi 固件路径问题已经解决。

继续检查网卡：

```bash
ip link
nmcli dev
```

如果 `wlan0` 能正常被识别，就可以重新启动 NetworkManager：

```bash
sudo systemctl start NetworkManager
```

扫描 Wi-Fi：

```bash
nmcli dev wifi list
```

连接 Wi-Fi：

```bash
sudo nmcli dev wifi connect "你的WiFi名称" password "你的WiFi密码"
```

连接后检查 IP：

```bash
ip addr show wlan0
```

测试网络：

```bash
ping -c 4 8.8.8.8
ping -c 4 www.baidu.com
```

如果能拿到 IP，并且 ping 通，Wi-Fi 修复完成。

最后因为 rootfs 镜像烧录时为了兼容工具做得比较小，可以在板端扩容：

```bash
sudo resize2fs /dev/mmcblk0p6
df -h /
```

这样 rootfs 会扩展到 eMMC 分区实际大小。

---

## **十五、这次经历学到的东西**

这次修 Wi-Fi 最大的收获是：不要一看到 Wi-Fi 起不来就马上怀疑 DTS 或驱动。应该先从日志判断驱动卡在哪个阶段。

这次的真实链路是：

```text
wlan0 存在
↓
手动 ip link set wlan0 up 失败
↓
dmesg 显示 bcmdhd 进入 wl_android_wifi_on
↓
Wi-Fi GPIO 上电成功
↓
SDIO 重新枚举成功
↓
驱动准备下载 firmware
↓
打开 /vendor/etc/firmware/fw_bcm43438a1.bin 失败
↓
确认 /vendor 根本不存在
↓
把 AP6212A1 固件固化进 Ubuntu rootfs
↓
重新制作 rootfs.img
↓
整包 update.img 加载失败
↓
改用 RKDevTool 分区烧录
↓
下载成功
```

最终问题不是一个“高级驱动 bug”，而是一个典型的移植问题：**Rockchip 原始驱动期望 Android/Buildroot 风格的 firmware 路径，但 Ubuntu rootfs 没有对应目录。**

真正的修复点只有一个：

```text
/vendor/etc/firmware/
```

里面至少需要：

```text
fw_bcm43438a1.bin
nvram_ap6212a.txt
config.txt
```

而真正稳定的烧录方式是：

```text
不要依赖超大 update.img 整包加载；
必要时直接使用 RKDevTool 分区镜像下载。
```

---
