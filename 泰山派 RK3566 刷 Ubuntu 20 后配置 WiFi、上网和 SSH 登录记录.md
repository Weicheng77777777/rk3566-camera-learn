### **泰山派 RK3566 刷 Ubuntu 20 后配置 WiFi、上网和 SSH 登录记录**

时间：2026-05-30  
设备：泰山派 RK3566  
系统：Ubuntu 20.04 arm64  
连接方式：串口调试 + 电脑移动热点 + SSH

这次给泰山派 RK3566 重新刷了 Ubuntu 20.04。刷完之后，系统默认没有自动连 WiFi，也没有一开始就能 SSH 登录，所以整个过程需要先通过串口手动把网络拉起来，再安装和配置 SSH 服务。

最终实现的目标是：

```text
泰山派连接电脑热点
泰山派可以通过电脑热点上网
电脑可以通过 SSH 登录泰山派
后续可以把 WiFi 配置写成开机自启动脚本
```

### **一、网络环境**

电脑开启移动热点，热点信息如下：

```text
SSID：weicheng
密码：88888888
```

电脑热点对应的虚拟网卡是 Windows 的 Wi-Fi Direct 适配器。任务管理器中可以看到类似：

```text
Wi-Fi Direct
Microsoft Wi-Fi Direct Virtual Adapter
SSID：weicheng
IPv4 地址：192.168.137.2
```

这里要特别注意，电脑有两个网络角色：

```text
电脑自己的上网网卡：WLAN，地址可能是 10.x.x.x
电脑热点虚拟网卡：Wi-Fi Direct，地址是 192.168.137.2
```

泰山派连接的是电脑热点，所以泰山派的网关应该写电脑热点虚拟网卡的地址，也就是：

```text
网关：192.168.137.2
```

不能写电脑上级网络的网关，也不能随便写 `192.168.1.1`。

最终采用的静态网络配置是：

```text
电脑热点 IP：192.168.137.2
泰山派 wlan0：192.168.137.222/24
默认网关：192.168.137.2
DNS：223.5.5.5 / 114.114.114.114
```

### **二、手动连接 WiFi**

刷机后先通过串口进入系统，确认无线网卡名称：

```sh
ip link
```

确认无线网卡是：

```text
wlan0
```

创建 WiFi 配置目录：

```sh
mkdir -p /etc/wifi
mkdir -p /var/run/wpa_supplicant
```

写入 `wpa_supplicant` 配置：

```sh
cat > /etc/wifi/wpa_supplicant.conf <<'EOF'
ctrl_interface=/var/run/wpa_supplicant
update_config=1

network={
    ssid="weicheng"
    psk="88888888"
    key_mgmt=WPA-PSK
}
EOF
```

重置无线网卡并启动连接：

```sh
killall wpa_supplicant 2>/dev/null

ip addr flush dev wlan0 2>/dev/null
ip link set wlan0 down
sleep 2
ip link set wlan0 up
sleep 5

wpa_supplicant -B -i wlan0 -Dnl80211 -c /etc/wifi/wpa_supplicant.conf
sleep 8
```

如果串口日志里出现类似下面的内容，说明 WiFi 认证已经成功：

```text
wl_cfg80211_connect : Connecting with ...
wl_notify_connect_status : wl_bss_connect_done succeeded
wl_iw_event : Link UP
```

这表示 WiFi 已经连上热点，但此时还不一定有 IP。

### **三、配置静态 IP、网关和 DNS**

这次系统里没有 `udhcpc`，所以不能通过原来的 DHCP 命令自动拿 IP：

```sh
udhcpc -i wlan0
```

执行后报错：

```text
-bash: udhcpc: command not found
```

因此改用静态 IP。

配置泰山派 IP：

```sh
ip addr flush dev wlan0
ip addr add 192.168.137.222/24 dev wlan0
```

配置默认网关：

```sh
ip route del default 2>/dev/null
ip route add default via 192.168.137.2 dev wlan0
```

配置 DNS：

```sh
echo "nameserver 223.5.5.5" > /etc/resolv.conf
echo "nameserver 114.114.114.114" >> /etc/resolv.conf
```

检查配置：

```sh
ip addr show wlan0
ip route
cat /etc/resolv.conf
```

正常应该看到：

```text
inet 192.168.137.222/24
default via 192.168.137.2 dev wlan0
```

### **四、测试网络是否正常**

先测试泰山派到电脑热点网关：

```sh
ping -c 4 192.168.137.2
```

如果能通，说明泰山派和电脑热点之间已经通信正常。

再测试外网 IP：

```sh
ping -c 4 223.5.5.5
```

如果能通，说明泰山派已经可以通过电脑热点上网。

最后测试域名解析：

```sh
ping -c 4 baidu.com
```

如果 `223.5.5.5` 能通，但 `baidu.com` 不通，一般是 DNS 问题，需要重新写 `/etc/resolv.conf`。

这次最终测试结果是：

```text
ping 192.168.137.2 成功
ping 223.5.5.5 成功
ping baidu.com 成功
```

说明 WiFi、网关、DNS 和上网都已经正常。

### **五、安装 SSH 服务**

刚刷完系统时，检查发现系统里没有 SSH 服务端：

```sh
which sshd
which dropbear
find / -name sshd 2>/dev/null
find / -name dropbear 2>/dev/null
```

都没有输出。

这说明系统里没有安装 `sshd` 或 `dropbear`，所以此时电脑无法 SSH 登录。即使网络通了，MobaXterm 也会提示连接失败或拒绝连接。

先确认系统有 `apt`：

```sh
which apt
which apt-get
```

系统中确实有：

```text
/usr/bin/apt
/usr/bin/apt-get
```

然后更新软件源：

```sh
apt update
```

### **六、处理 apt 损坏包问题**

安装 SSH 时一开始失败：

```sh
apt install -y openssh-server
```

报错：

```text
E: dpkg was interrupted, you must manually run 'dpkg --configure -a' to correct the problem.
```

先修复：

```sh
dpkg --configure -a
apt --fix-broken install -y
```

之后又遇到一个坏包：

```text
E: The package libmali-bifrost-g52-g2p0-x11 needs to be reinstalled, but I can't find an archive for it.
```

这个包是 Mali GPU 相关库，不是 SSH 必需包。它处于损坏状态，并且当前软件源里找不到对应安装包，导致 `apt` 被卡住。

强制移除：

```sh
dpkg --remove --force-remove-reinstreq libmali-bifrost-g52-g2p0-x11
```

然后继续修复：

```sh
apt --fix-broken install -y
dpkg --configure -a
apt update
```

这一步之后，`libmali` 坏包不再阻塞 `apt`。

### **七、处理 OpenSSH 版本依赖冲突**

继续安装：

```sh
apt install -y openssh-server
```

又出现依赖版本冲突：

```text
openssh-server : Depends: openssh-client (= 1:8.2p1-4ubuntu0.13) but 1:8.2p1-4ubuntu0.9 is to be installed
                 Depends: openssh-sftp-server but it is not going to be installed
E: Unable to correct problems, you have held broken packages.
```

问题原因是 `openssh-server`、`openssh-client`、`openssh-sftp-server` 版本必须一致，但系统里准备安装的版本不一致。

先查看可用版本：

```sh
apt-cache policy openssh-client openssh-server openssh-sftp-server
```

然后安装统一版本，例如：

```sh
apt install -y openssh-client=1:8.2p1-4ubuntu0.13 openssh-sftp-server=1:8.2p1-4ubuntu0.13 openssh-server=1:8.2p1-4ubuntu0.13
```

如果版本号不同，要根据 `apt-cache policy` 的输出选择三个包都有的同一个版本。

安装成功后检查：

```sh
which sshd
```

正常应该输出：

```text
/usr/sbin/sshd
```

### **八、启动 SSH 服务**

生成 SSH 主机密钥：

```sh
ssh-keygen -A
```

创建运行目录：

```sh
mkdir -p /run/sshd
```

启动 SSH：

```sh
service ssh restart
```

如果 `service` 不可用，可以直接启动：

```sh
/usr/sbin/sshd
```

检查 22 端口：

```sh
ss -lntp | grep :22
```

或者：

```sh
netstat -tnlp | grep :22
```

如果看到类似：

```text
LISTEN 0 128 0.0.0.0:22
```

说明 SSH 服务已经启动。

### **九、root 密码正确也可能 SSH 登录不上**

这是这次折腾里最容易误判的一点。

一开始 MobaXterm 已经能连到：

```text
192.168.137.222
```

但是输入 root 密码后提示：

```text
Access denied
```

这说明网络已经通了，SSH 服务也已经响应了，但登录认证失败。

这里不一定是 root 密码错，也可能是 SSH 默认配置不允许 root 通过密码登录。

先在串口里重新设置 root 密码：

```sh
passwd root
```

然后修改 SSH 配置：

```sh
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config

sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config

grep -q "^PermitRootLogin" /etc/ssh/sshd_config || echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
grep -q "^PasswordAuthentication" /etc/ssh/sshd_config || echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
```

检查配置：

```sh
grep -n "PermitRootLogin\|PasswordAuthentication" /etc/ssh/sshd_config
```

应该看到：

```text
PermitRootLogin yes
PasswordAuthentication yes
```

重启 SSH：

```sh
service ssh restart
```

如果不行：

```sh
killall sshd 2>/dev/null
mkdir -p /run/sshd
/usr/sbin/sshd
```

然后重新用 MobaXterm 登录：

```text
Host：192.168.137.222
User：root
Port：22
Password：刚才 passwd root 设置的密码
```

这次就可以正常登录。

关键结论：

```text
root 密码正确，不代表一定能 SSH 登录。
如果 sshd_config 禁止 root 登录或禁止密码登录，仍然会 Access denied。
```

### **十、写入 WiFi 开机自启动脚本**

前面所有 WiFi 和 IP 配置都是手动执行的，重启后会丢失。所以需要写一个启动脚本。

创建脚本：

```sh
cat > /etc/init.d/S42wifi_sta <<'EOF'
#!/bin/sh
#
# S42wifi_sta
# Auto connect wlan0 to PC hotspot on boot.
#

WLAN_IF="wlan0"
WIFI_DIR="/etc/wifi"
WPA_CONF="/etc/wifi/wpa_supplicant.conf"

SSID="weicheng"
PSK="88888888"

STATIC_IP="192.168.137.222/24"
GATEWAY="192.168.137.2"

DNS1="223.5.5.5"
DNS2="114.114.114.114"

start_wifi()
{
    echo "[wifi] start WiFi auto connect..."

    mkdir -p "$WIFI_DIR"
    mkdir -p /var/run/wpa_supplicant

    cat > "$WPA_CONF" <<CONFEOF
ctrl_interface=/var/run/wpa_supplicant
update_config=1

network={
    ssid="$SSID"
    psk="$PSK"
    key_mgmt=WPA-PSK
}
CONFEOF

    echo "[wifi] stop old processes..."
    killall wpa_supplicant 2>/dev/null
    killall dhclient 2>/dev/null
    killall dhcpcd 2>/dev/null

    echo "[wifi] reset wlan0..."
    ip addr flush dev "$WLAN_IF" 2>/dev/null
    ip link set "$WLAN_IF" down 2>/dev/null
    sleep 2
    ip link set "$WLAN_IF" up 2>/dev/null

    echo "[wifi] wait wlan0 ready..."
    sleep 5

    echo "[wifi] connect to SSID: $SSID"
    wpa_supplicant -B -i "$WLAN_IF" -Dnl80211 -c "$WPA_CONF"

    echo "[wifi] wait association..."
    sleep 8

    echo "[wifi] set static IP..."
    ip addr flush dev "$WLAN_IF" 2>/dev/null
    ip addr add "$STATIC_IP" dev "$WLAN_IF"

    echo "[wifi] set default gateway..."
    ip route del default 2>/dev/null
    ip route add default via "$GATEWAY" dev "$WLAN_IF"

    echo "[wifi] set DNS..."
    echo "nameserver $DNS1" > /etc/resolv.conf
    echo "nameserver $DNS2" >> /etc/resolv.conf

    echo "[wifi] wlan0 status:"
    ip addr show "$WLAN_IF"

    echo "[wifi] route table:"
    ip route

    echo "[wifi] DNS:"
    cat /etc/resolv.conf 2>/dev/null
}

stop_wifi()
{
    echo "[wifi] stop WiFi..."

    killall wpa_supplicant 2>/dev/null
    killall dhclient 2>/dev/null
    killall dhcpcd 2>/dev/null

    ip addr flush dev "$WLAN_IF" 2>/dev/null
    ip link set "$WLAN_IF" down 2>/dev/null
}

case "$1" in
    start)
        start_wifi
        ;;
    stop)
        stop_wifi
        ;;
    restart)
        stop_wifi
        sleep 2
        start_wifi
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac

exit 0
EOF
```

赋予执行权限：

```sh
chmod +x /etc/init.d/S42wifi_sta
sync
```

手动测试：

```sh
/etc/init.d/S42wifi_sta restart
```

测试网络：

```sh
ping -c 4 192.168.137.2
ping -c 4 223.5.5.5
ping -c 4 baidu.com
```

### **十一、如果系统是 systemd，需要添加服务**

Ubuntu 20 通常使用 systemd。先确认：

```sh
ps -p 1 -o comm=
```

如果输出是：

```text
systemd
```

则创建 systemd 服务：

```sh
cat > /etc/systemd/system/wifi-sta.service <<'EOF'
[Unit]
Description=WiFi STA Auto Connect
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/etc/init.d/S42wifi_sta start
ExecStop=/etc/init.d/S42wifi_sta stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

启用服务：

```sh
systemctl daemon-reload
systemctl enable wifi-sta.service
systemctl start wifi-sta.service
sync
```

查看状态：

```sh
systemctl status wifi-sta.service
```

### **十二、最终验证**

重启开发板：

```sh
reboot
```

等待系统启动后，在电脑上测试：

```bat
ping 192.168.137.222
```

如果能 ping 通，再用 SSH 登录：

```bat
ssh root@192.168.137.222
```

或者 MobaXterm：

```text
Remote host：192.168.137.222
Username：root
Port：22
```

开发板上最终应满足：

```sh
ip addr show wlan0
```

能看到：

```text
inet 192.168.137.222/24
```

执行：

```sh
ip route
```

能看到：

```text
default via 192.168.137.2 dev wlan0
```

执行：

```sh
ping -c 4 baidu.com
```

可以正常返回。

### **十三、这次踩坑总结**

这次主要踩了几个坑。

第一个坑是把电脑的 WLAN 网关和电脑热点网关搞混了。电脑自己的 WLAN 地址是 `10.x.x.x`，但泰山派连接的是电脑移动热点，所以网关应该是 Wi-Fi Direct 虚拟网卡地址 `192.168.137.2`。

第二个坑是系统没有 `udhcpc`，所以不能直接 DHCP 获取 IP。手动配置静态 IP 更稳定，也方便后续 SSH。

第三个坑是刚刷完系统没有 SSH 服务端。没有 `sshd` 或 `dropbear` 时，电脑不可能通过 SSH 登录开发板。

第四个坑是 `apt` 被坏掉的 `libmali-bifrost-g52-g2p0-x11` 包卡住，需要先强制移除损坏包。

第五个坑是 `openssh-server` 和 `openssh-client` 版本不一致，必须让 `openssh-client`、`openssh-sftp-server`、`openssh-server` 安装同一个版本。

第六个坑是 root 密码正确也不一定能 SSH 登录。如果 `/etc/ssh/sshd_config` 里没有允许 root 登录和密码登录，MobaXterm 仍然会提示 `Access denied`。

最终关键配置是：

```text
SSID：weicheng
PSK：88888888
泰山派 IP：192.168.137.222/24
电脑热点网关：192.168.137.2
DNS：223.5.5.5 / 114.114.114.114
SSH 用户：root
SSH 端口：22
```

