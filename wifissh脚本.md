## **一、先说最推荐的做法**

Ubuntu 22.04 本身就支持：

- NetworkManager 自动连接 Wi‑Fi
- `ssh` 服务开机自启

所以你真正要做的是这几步：

```bash
1. 用 nmcli 连接一次你的 Wi‑Fi，并保存连接
2. 打开这个连接的 autoconnect
3. 安装并启用 openssh-server
4. 设置 ssh 服务开机自启
```

---

## **二、先手动保存 Wi‑Fi 连接**

你日志里已经扫到：

```text
weicheng
```

所以先执行：

```bash
sudo nmcli dev wifi connect "weicheng" password "88888888"
```

如果成功，NetworkManager 会自动创建一个连接配置。

然后查看连接列表：

```bash
nmcli connection show
```

你应该能看到一个类似：

```text
weicheng
```

接着把它设置为开机自动连接：

```bash
sudo nmcli connection modify "weicheng" connection.autoconnect yes
```

为了避免 wlan0 还没完全准备好，也可以顺手设一下连接优先级：

```bash
sudo nmcli connection modify "weicheng" connection.autoconnect-priority 10
```

再确认：

```bash
nmcli connection show "weicheng" | grep autoconnect
```

你希望看到类似：

```text
connection.autoconnect: yes
connection.autoconnect-priority: 10
```

---

## **三、安装并启用 SSH**

从你的日志看：

```text
Started OpenBSD Secure Shell server.
```

这说明 `openssh-server` 大概率已经装了。

先确认：

```bash
systemctl status ssh --no-pager
```

如果服务存在，再执行：

```bash
sudo systemctl enable ssh
sudo systemctl restart ssh
```

再确认：

```bash
systemctl is-enabled ssh
systemctl is-active ssh
```

你希望看到：

```text
enabled
active
```

如果 `ssh` 服务不存在，就先安装：

```bash
sudo apt update
sudo apt install -y openssh-server
```

然后再启用：

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## **四、如果你只想要“开机自动连 Wi‑Fi + 开机启 SSH”，到这里其实已经够了**

最简完成版命令就是：

```bash
sudo nmcli dev wifi connect "weicheng" password "88888888"
sudo nmcli connection modify "weicheng" connection.autoconnect yes
sudo nmcli connection modify "weicheng" connection.autoconnect-priority 10

sudo systemctl enable ssh
sudo systemctl restart ssh
```

然后重启测试：

```bash
sudo reboot
```

开机后登录串口检查：

```bash
nmcli dev status
ip addr show wlan0
systemctl status ssh --no-pager
```

如果看到：

- `wlan0` 已连接
- `ssh` 是 active

那就已经实现目标了。

---

## **五、如果你想要一个“更保险的开机脚本”**

虽然标准方式已经够用，但你说“来一个开机脚本”，那我给你一个 **systemd oneshot 自检脚本**。  
它做三件事：

1. 等待 NetworkManager 就绪  
2. 如果 `weicheng` 没连上，就主动拉起连接  
3. 确保 ssh 服务已启动

这个方案比较适合嵌入式板子调试。

---

## **六、创建开机 Wi‑Fi + SSH 自检脚本**

先创建脚本文件：

```bash
sudo nano /usr/local/bin/auto-wifi-ssh.sh
```

把下面内容完整粘进去：

```bash
#!/bin/bash
set -e

LOG_FILE="/var/log/auto-wifi-ssh.log"
SSID="weicheng"
IFACE="wlan0"

echo "==== $(date '+%F %T') auto-wifi-ssh start ====" >> "$LOG_FILE"

# 等待 NetworkManager 启动
for i in $(seq 1 20); do
    if systemctl is-active --quiet NetworkManager; then
        echo "$(date '+%F %T') NetworkManager is active" >> "$LOG_FILE"
        break
    fi
    sleep 1
done

# 等待无线网卡出现
for i in $(seq 1 20); do
    if ip link show "$IFACE" >/dev/null 2>&1; then
        echo "$(date '+%F %T') Interface $IFACE found" >> "$LOG_FILE"
        break
    fi
    sleep 1
done

# 如果还没连接 Wi-Fi，则尝试连接已保存的连接
CURRENT_SSID="$(nmcli -t -f ACTIVE,SSID dev wifi 2>/dev/null | grep '^yes:' | cut -d: -f2- || true)"
if [ "$CURRENT_SSID" != "$SSID" ]; then
    echo "$(date '+%F %T') Wi-Fi not connected to $SSID, trying nmcli up" >> "$LOG_FILE"
    nmcli connection up "$SSID" >> "$LOG_FILE" 2>&1 || true
else
    echo "$(date '+%F %T') Wi-Fi already connected to $SSID" >> "$LOG_FILE"
fi

# 再等几秒让 DHCP 完成
sleep 5

# 启动并确保 ssh 运行
systemctl start ssh >> "$LOG_FILE" 2>&1 || true

if systemctl is-active --quiet ssh; then
    echo "$(date '+%F %T') ssh is active" >> "$LOG_FILE"
else
    echo "$(date '+%F %T') ssh failed to start" >> "$LOG_FILE"
fi

IP_ADDR="$(ip -4 addr show "$IFACE" | awk '/inet /{print $2}' | head -n1 || true)"
echo "$(date '+%F %T') $IFACE ip=$IP_ADDR" >> "$LOG_FILE"

echo "==== $(date '+%F %T') auto-wifi-ssh end ====" >> "$LOG_FILE"
```

保存退出后，赋予执行权限：

```bash
sudo chmod +x /usr/local/bin/auto-wifi-ssh.sh
```

---

## **七、创建 systemd 服务**

创建服务文件：

```bash
sudo nano /etc/systemd/system/auto-wifi-ssh.service
```

粘贴下面内容：

```ini
[Unit]
Description=Auto connect WiFi and ensure SSH is running
After=NetworkManager.service ssh.service
Wants=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/auto-wifi-ssh.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

保存退出后执行：

```bash
sudo systemctl daemon-reload
sudo systemctl enable auto-wifi-ssh.service
sudo systemctl start auto-wifi-ssh.service
```

检查服务状态：

```bash
systemctl status auto-wifi-ssh.service --no-pager
```

看日志：

```bash
cat /var/log/auto-wifi-ssh.log
```

---

## **八、完整推荐命令，一次配好**

如果你想一口气做完，按下面顺序执行：

```bash
sudo nmcli dev wifi connect "weicheng" password "88888888"
sudo nmcli connection modify "weicheng" connection.autoconnect yes
sudo nmcli connection modify "weicheng" connection.autoconnect-priority 10

sudo systemctl enable NetworkManager
sudo systemctl restart NetworkManager

sudo systemctl enable ssh
sudo systemctl restart ssh
```

然后再创建脚本和服务：

```bash
sudo tee /usr/local/bin/auto-wifi-ssh.sh > /dev/null <<'EOF'
#!/bin/bash
set -e

LOG_FILE="/var/log/auto-wifi-ssh.log"
SSID="weicheng"
IFACE="wlan0"

echo "==== $(date '+%F %T') auto-wifi-ssh start ====" >> "$LOG_FILE"

for i in $(seq 1 20); do
    if systemctl is-active --quiet NetworkManager; then
        echo "$(date '+%F %T') NetworkManager is active" >> "$LOG_FILE"
        break
    fi
    sleep 1
done

for i in $(seq 1 20); do
    if ip link show "$IFACE" >/dev/null 2>&1; then
        echo "$(date '+%F %T') Interface $IFACE found" >> "$LOG_FILE"
        break
    fi
    sleep 1
done

CURRENT_SSID="$(nmcli -t -f ACTIVE,SSID dev wifi 2>/dev/null | grep '^yes:' | cut -d: -f2- || true)"
if [ "$CURRENT_SSID" != "$SSID" ]; then
    echo "$(date '+%F %T') Wi-Fi not connected to $SSID, trying nmcli up" >> "$LOG_FILE"
    nmcli connection up "$SSID" >> "$LOG_FILE" 2>&1 || true
else
    echo "$(date '+%F %T') Wi-Fi already connected to $SSID" >> "$LOG_FILE"
fi

sleep 5

systemctl start ssh >> "$LOG_FILE" 2>&1 || true

if systemctl is-active --quiet ssh; then
    echo "$(date '+%F %T') ssh is active" >> "$LOG_FILE"
else
    echo "$(date '+%F %T') ssh failed to start" >> "$LOG_FILE"
fi

IP_ADDR="$(ip -4 addr show "$IFACE" | awk '/inet /{print $2}' | head -n1 || true)"
echo "$(date '+%F %T') $IFACE ip=$IP_ADDR" >> "$LOG_FILE"

echo "==== $(date '+%F %T') auto-wifi-ssh end ====" >> "$LOG_FILE"
EOF

sudo chmod +x /usr/local/bin/auto-wifi-ssh.sh
```

再创建服务：

```bash
sudo tee /etc/systemd/system/auto-wifi-ssh.service > /dev/null <<'EOF'
[Unit]
Description=Auto connect WiFi and ensure SSH is running
After=NetworkManager.service ssh.service
Wants=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/auto-wifi-ssh.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable auto-wifi-ssh.service
sudo systemctl start auto-wifi-ssh.service
```

---

## **九、如何验证开机自动连接成功**

先手动看当前状态：

```bash
nmcli dev status
ip addr show wlan0
systemctl status ssh --no-pager
```

然后重启测试：

```bash
sudo reboot
```

重启后串口登录，执行：

```bash
nmcli dev status
nmcli connection show --active
ip addr show wlan0
systemctl is-active ssh
systemctl status auto-wifi-ssh.service --no-pager
cat /var/log/auto-wifi-ssh.log
```

你希望看到：

- `wlan0` 已连接到 `weicheng`
- `wlan0` 拿到了 IP
- `ssh` 是 `active`
- `auto-wifi-ssh.service` 成功执行

---

## **十、如果你后面换 Wi‑Fi 名字，只改一个地方**

如果以后你改成别的热点，比如：

```text
baiyun
```

你只需要改两处之一：

### 标准方案
重新连接一次：

```bash
sudo nmcli dev wifi connect "baiyun" password "你的密码"
sudo nmcli connection modify "baiyun" connection.autoconnect yes
```

### 自检脚本方案
改脚本里的：

```bash
SSID="weicheng"
```

改成：

```bash
SSID="baiyun"
```

然后：

```bash
sudo systemctl restart auto-wifi-ssh.service
```

---

## **十一、我对你当前板子的建议**

你现在这个系统已经比较稳定了，我建议最终采用下面这个组合：

- 用 **NetworkManager 保存 Wi‑Fi**
- 用 **`systemctl enable ssh`** 保证 SSH 开机启动
- 保留 **`auto-wifi-ssh.service`** 作为保险层

这样后面即使 NetworkManager 某次没自动拉起到指定 SSID，这个脚本也会补一刀。
