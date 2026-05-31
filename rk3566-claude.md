# 泰山派 RK3566 Ubuntu 22.04：网络、SSH、Claude Code + DeepSeek 跑通记录

日期：2026-05-31  
板子：立创泰山派 RK3566  
系统：Ubuntu 22.04 LTS  
内核：Linux 4.19.232 aarch64  
登录用户：ubuntu  
最终状态：USB 网络共享 + SSH 已跑通，Claude Code 已通过 npm 安装路线跑通，并准备接入 DeepSeek API。

---

## 一、直接跟着答案走

这一部分是最终可复现流程。如果你不想看踩坑过程，直接按这一节做。

---

## 1. 先确认板子能联网并能 SSH

最终我们使用的是电脑给开发板共享网络，开发板拿到的 IP 类似：

```bash
192.168.137.97
```

Windows 电脑通过 MobaXterm SSH 登录：

```bash
ssh ubuntu@192.168.137.97
```

登录成功后会看到类似：

```text
Welcome to Ubuntu 22.04 LTS (GNU/Linux 4.19.232 aarch64)
ubuntu@tspi-ubuntu22:~$
```

到这里说明 SSH 已经通了，后续全部在 SSH 终端里操作即可。

---

## 2. 开发板安装 Node.js

Claude Code 的 npm 版本需要 Node.js。板子上最开始没有 npm：

```bash
npm: command not found
```

所以先在开发板上安装 Node.js。

如果网络正常，可以执行：

```bash
curl -fsSL <NodeSource Node.js 20 setup script URL> | sudo -E bash -
sudo apt-get install -y nodejs
```

安装完成后检查：

```bash
node -v
npm -v
```

实际跑通时 Node.js 版本为：

```text
Node.js v20.20.2
```

只要能看到 Node.js 和 npm 版本，就可以继续。

---

## 3. 清理旧版 Claude Code

之前手动放进去的 Claude Code 二进制版会崩溃，所以必须先删掉。

执行：

```bash
sudo npm uninstall -g @anthropic-ai/claude-code
sudo rm -f /usr/local/bin/claude
sudo rm -f /usr/bin/claude
sudo rm -rf /usr/lib/node_modules/@anthropic-ai/claude-code
sudo rm -rf /usr/local/lib/node_modules/@anthropic-ai/claude-code
hash -r
```

如果提示某些文件不存在，不用管，继续下一步。

---

## 4. 回到用户主目录

这是一个关键点。

如果你刚刚删掉了当前所在目录，再运行 npm 会报：

```text
Error: ENOENT: no such file or directory, uv_cwd
```

所以安装前一定先执行：

```bash
cd ~
```

确认当前目录安全：

```bash
pwd
```

应该看到：

```bash
/home/ubuntu
```

---

## 5. 用 npm 镜像安装 Claude Code 指定版本

最终推荐安装指定版本，例如：

```bash
sudo npm install -g @anthropic-ai/claude-code@2.1.154 --registry=<npm mirror URL>
```

如果网络足够好，也可以直接使用官方 npm 源：

```bash
sudo npm install -g @anthropic-ai/claude-code@2.1.154
```

安装成功时会看到类似：

```text
added 1 package
```

然后检查命令位置：

```bash
which claude
```

正常应该输出类似：

```bash
/usr/bin/claude
```

检查版本：

```bash
claude --version
```

如果没有再出现 `Bun has crashed`、`Bus error` 或 `claude native binary not installed`，说明安装路线已经正确。

---

## 6. 配置 Claude Code 接入 DeepSeek

把 DeepSeek 的 API 配置写入 `~/.bashrc`。

注意：这里的 Key 必须替换成你自己的真实 DeepSeek API Key。

```bash
cat >> ~/.bashrc << 'EOF'

# Claude Code + DeepSeek
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-你的真实DeepSeek-API-Key"
export ANTHROPIC_MODEL="deepseek-chat"
EOF

source ~/.bashrc
```

检查是否生效：

```bash
echo $ANTHROPIC_BASE_URL
echo $ANTHROPIC_MODEL
```

应该看到：

```text
https://api.deepseek.com/anthropic
deepseek-chat
```

---

## 7. 启动 Claude Code

执行：

```bash
claude
```

第一次启动会进入初始化界面，让你选择主题。

推荐直接选择：

```text
1. Auto
```

或者：

```text
2. Dark mode
```

按回车确认即可。

进入 Claude Code 后，可以先输入：

```text
/status
```

或者直接让它检查板子：

```text
我是立创泰山派 RK3566，运行 Ubuntu 22.04，内核 4.19.232。请帮我检查当前系统状态，包括 CPU、内存、网络、SSH、/dev/fb0 显示设备。
```

---

## 8. 模型名建议

这次踩坑里最容易混淆的是模型名。

不建议直接写：

```bash
deepseek-v4-flash
```

因为 Claude Code 可能提示：

```text
There's an issue with the selected model deepseek-v4-flash. It may not exist or you may not have access to it.
```

更稳的写法是：

```bash
export ANTHROPIC_MODEL="deepseek-chat"
```

如果后续 DeepSeek 官方接口确认支持其他模型名，再单独替换。

---

## 9. 最终推荐的一键配置块

如果 Node.js 已经装好，并且旧版已经清理完，可以直接用下面这套：

```bash
cd ~

sudo npm uninstall -g @anthropic-ai/claude-code
sudo rm -f /usr/local/bin/claude
sudo rm -f /usr/bin/claude
sudo rm -rf /usr/lib/node_modules/@anthropic-ai/claude-code
sudo rm -rf /usr/local/lib/node_modules/@anthropic-ai/claude-code
hash -r

sudo npm install -g @anthropic-ai/claude-code@2.1.154 --registry=<npm mirror URL>

cat >> ~/.bashrc << 'EOF'

# Claude Code + DeepSeek
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-你的真实DeepSeek-API-Key"
export ANTHROPIC_MODEL="deepseek-chat"
EOF

source ~/.bashrc

which claude
claude --version
claude
```

---

# 二、我的踩坑日记

这一部分记录完整排错过程，方便以后复盘。

---

## 1. 一开始板子没有 npm

最初在板子上执行：

```bash
claude --version
```

结果：

```text
bash: claude: command not found
```

然后尝试：

```bash
npm install -g @anthropic-ai/claude-code
```

结果：

```text
bash: npm: command not found
```

说明 Ubuntu rootfs 里没有 Node.js/npm 环境。

结论：不能直接装 Claude Code，必须先解决 Node.js/npm。

---

## 2. 尝试下载 DeepSeek-TUI ARM64 二进制

后来考虑不用 Claude Code，直接下载 DeepSeek-TUI 的 ARM64 二进制文件。

文件名类似：

```text
deepseek-linux-arm64
deepseek-tui-linux-arm64
```

上传到板子后执行：

```bash
chmod +x deepseek-linux-arm64 deepseek-tui-linux-arm64
mv deepseek-linux-arm64 deepseek
mv deepseek-tui-linux-arm64 deepseek-tui
sudo mv deepseek deepseek-tui /usr/local/bin/
```

但是运行时报错：

```text
/lib/aarch64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
/lib/aarch64-linux-gnu/libc.so.6: version `GLIBC_2.39' not found
```

板子系统是 Ubuntu 22.04，glibc 是 2.35。  
这个 DeepSeek-TUI 二进制要求 glibc 2.38/2.39，所以跑不起来。

结论：DeepSeek-TUI 当前下载到的预编译版本不适合这个 Ubuntu 22.04 rootfs，除非重新找兼容 glibc 2.35 的版本，或者自己编译。

---

## 3. 尝试 Claude Code ARM64 二进制包

接着下载 Claude Code 的 Linux ARM64 tar 包，解压后把 `claude` 放到：

```bash
/usr/local/bin/claude
```

执行：

```bash
claude --version
```

结果出现：

```text
Bun v1.3.14 Linux arm64
Linux Kernel v4.19.232 | glibc v2.35
CPU: neon fp aes crc32 atomics

panic(main thread): Bus error at address ...
oh no: Bun has crashed. This indicates a bug in Bun, not your code.

Trace/breakpoint trap
```

这个错误非常关键。

它说明这个 Claude Code 二进制版内部使用了 Bun 运行时，而 Bun 在 RK3566 + Linux 4.19.232 + glibc 2.35 这个组合上发生了底层崩溃。

结论：不能使用这个直接下载的 Claude Code ARM64 二进制包。即使版本旧一点，也可能继续触发 Bun 的 Bus error。

---

## 4. 尝试下载 claude-code-2.1.154.tgz 后本地安装

为了避开二进制版，尝试下载 npm 包：

```text
claude-code-2.1.154.tgz
```

然后上传到板子，执行：

```bash
tar -xzvf claude-code-2.1.154.tgz
cd package
sudo npm install -g .
```

结果报错：

```text
ERROR: Direct publishing is not allowed.
Please see the release workflow documentation to publish this package.
```

说明这个 npm 包里有安装保护脚本，不允许直接从解压目录执行 `npm install -g .`。

结论：不能直接在解压后的 package 目录里 `npm install -g .`。

---

## 5. 尝试加 --ignore-scripts

为了跳过安装脚本，执行：

```bash
sudo npm install -g ./claude-code-2.1.154.tgz --ignore-scripts
```

这次显示：

```text
added 1 package
```

看起来成功了。

但是运行：

```bash
claude
```

结果报错：

```text
Error: claude native binary not installed.

Either postinstall did not run (--ignore-scripts, some npm configs)
or the platform-native optional dependency was not downloaded (--omit=optional).

Run the postinstall manually:
node node_modules/@anthropic-ai/claude-code/install.cjs
```

原因很明确：  
`--ignore-scripts` 跳过了 postinstall，而 Claude Code 正是靠 postinstall 安装原生运行组件。

结论：`--ignore-scripts` 虽然绕过了 Direct publishing 检查，但也导致 Claude Code 缺核心组件，不能用。

---

## 6. 尝试手动补 binary，但是文件根本不存在

根据错误提示，尝试进入：

```bash
/usr/lib/node_modules/@anthropic-ai/claude-code/
```

然后创建 bin 目录：

```bash
sudo mkdir -p bin
```

尝试复制：

```bash
sudo cp /home/ubuntu/package/claude ./bin/claude
```

结果：

```text
cp: cannot stat '/home/ubuntu/package/claude': No such file or directory
```

再搜索：

```bash
find /home/ubuntu -name "claude"
```

没有找到任何 `claude` 文件。

这说明 `.tgz` 解压出来的 npm 包里并没有我们以为的原生 `claude` 二进制文件。  
所以手动复制这条路走不通。

结论：不能靠从 package 目录里复制 `claude` 来修复。

---

## 7. 尝试直接 npm 官方源安装，网络超时

执行：

```bash
sudo npm install -g @anthropic-ai/claude-code@2.1.154
```

结果：

```text
npm error code ETIMEDOUT
npm error network request to registry.npmjs.org failed
```

这是网络问题。板子通过 USB 网络共享访问 npm 官方源不稳定。

结论：直接访问 npm 官方源不可靠，需要镜像源。

---

## 8. 使用 npm 镜像源时又遇到 uv_cwd

后面执行清理命令时，删除了当前所在目录：

```bash
sudo rm -rf /usr/lib/node_modules/@anthropic-ai/claude-code
```

但是当时终端正好就在这个目录里：

```bash
/usr/lib/node_modules/@anthropic-ai/claude-code
```

于是再运行 npm 安装时出现：

```text
Error: ENOENT: no such file or directory, uv_cwd
```

这个错误不是 npm 包的问题，而是当前工作目录已经被删除了。

解决方法非常简单：

```bash
cd ~
```

然后再执行 npm 安装。

结论：删除当前目录后，一定要先 `cd ~`，否则 npm 会因为找不到当前工作目录而崩溃。

---

## 9. 最终正确路线

最后正确路线是：

```bash
cd ~

sudo npm uninstall -g @anthropic-ai/claude-code
sudo rm -f /usr/local/bin/claude
sudo rm -f /usr/bin/claude
sudo rm -rf /usr/lib/node_modules/@anthropic-ai/claude-code
sudo rm -rf /usr/local/lib/node_modules/@anthropic-ai/claude-code
hash -r

sudo npm install -g @anthropic-ai/claude-code@2.1.154 --registry=<npm mirror URL>
```

然后配置 DeepSeek：

```bash
cat >> ~/.bashrc << 'EOF'

# Claude Code + DeepSeek
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-你的真实DeepSeek-API-Key"
export ANTHROPIC_MODEL="deepseek-chat"
EOF

source ~/.bashrc
```

最后启动：

```bash
claude
```

这条路线绕开了所有之前的坑：

1. 不用 DeepSeek-TUI 的高 glibc 二进制。
2. 不用 Claude Code 的 Bun 二进制包。
3. 不从解压 package 目录强行安装。
4. 不使用 `--ignore-scripts`。
5. 不依赖 npm 官方源。
6. 不在被删除的目录里运行 npm。
7. 使用板子本地 Node.js + npm 正规安装 Claude Code。

---

# 三、网络与 SSH 部分记录

## 1. Wi-Fi 状态

一开始板子 `wlan0` 是 UP 的，但是没有 IP：

```text
wlan0: flags=4099<UP,BROADCAST,MULTICAST>
```

没有看到：

```text
inet 192.168.x.x
```

说明 Wi-Fi 网卡存在，但没有拿到 IP。

执行：

```bash
sudo dhclient wlan0
```

时看到类似：

```text
[dhd-wlan0] wl_run_escan : LEGACY_SCAN
```

这是 Broadcom/AP6212 Wi-Fi 驱动在扫描热点。  
如果一直扫，说明 Wi-Fi 还没有真正连上热点。  
`dhclient` 只能在已经连接 Wi-Fi 的情况下获取 IP，不能负责 Wi-Fi 认证连接。

正确连接方式是：

```bash
sudo nmcli dev wifi connect "SSID名称" password "WiFi密码"
```

例如之前用过：

```bash
sudo nmcli dev wifi connect "weicheng" password "88888888"
```

---

## 2. USB 网络共享最终跑通

后来改成 USB 网络共享，Windows 电脑共享网络给开发板，开发板拿到了：

```text
192.168.137.97
```

MobaXterm 成功 SSH 登录：

```text
SSH session to ubuntu@192.168.137.97
Welcome to Ubuntu 22.04 LTS
```

这一步非常关键。  
只有 SSH 通了，后面传文件、装 Node.js、装 Claude Code 才方便。

---

## 3. Windows ICS 共享踩坑

Windows 开启 Internet 连接共享时遇到：

```text
无法启用 Internet 连接共享。为 LAN 连接配置的 IP 地址需要使用自动 IP 寻址。
```

这个错误通常和目标网卡的 IPv4 配置有关。

处理思路是：

1. 找到真正连接开发板的网卡。
2. 目标网卡 IPv4 改成自动获取 IP。
3. 关闭 WLAN 共享。
4. 重新打开 WLAN 共享。
5. 家庭网络连接选择正确的目标网卡。
6. 开发板执行 DHCP 获取 IP。

在开发板上可以执行：

```bash
sudo dhclient usb0
ifconfig usb0
```

如果 `usb0` 没有出现，则需要先用：

```bash
ifconfig -a
```

确认实际网卡名。

---

# 四、eDP 屏幕与 /dev/fb0 记录

板子底层已经识别到 eDP 屏幕，对应设备：

```bash
/dev/fb0
```

如果想把终端显示到 eDP 屏幕上，可以优先考虑 framebuffer 终端方案。

## 1. 安装 fbterm

```bash
sudo apt update
sudo apt install fbterm
```

## 2. 用户加入 video 组

```bash
sudo adduser ubuntu video
```

重新登录后生效。

## 3. 启动 framebuffer 终端

```bash
sudo fbterm
```

如果屏幕黑，先检查背光：

```bash
ls /sys/class/backlight/
```

如果存在背光设备，可以尝试：

```bash
echo 0 | sudo tee /sys/class/backlight/*/bl_power
```

或者调亮亮度：

```bash
cat /sys/class/backlight/*/max_brightness
echo 100 | sudo tee /sys/class/backlight/*/brightness
```

具体亮度值以实际 `max_brightness` 为准。

---

# 五、最终经验

## 1. ARM64 开发板不要随便用预编译二进制

这次两个工具都踩到了：

DeepSeek-TUI 报：

```text
GLIBC_2.38 not found
GLIBC_2.39 not found
```

Claude Code 二进制报：

```text
Bun has crashed
Bus error
```

这说明 ARM64 Linux 开发板上，预编译二进制经常受以下因素影响：

1. glibc 版本。
2. 内核版本。
3. CPU 指令集。
4. 运行时兼容性。
5. 编译环境是否比目标系统更新。

对 RK3566 + Ubuntu 22.04 + kernel 4.19 这种板子，最稳的是使用系统原生 Node.js/npm 路线。

---

## 2. 不要对 Claude Code 的 npm 包使用 --ignore-scripts

`--ignore-scripts` 会导致：

```text
claude native binary not installed
```

原因是 Claude Code 需要 postinstall 安装原生组件。  
跳过脚本就等于装了一个空壳。

---

## 3. 不能在解压后的 package 目录直接 npm install -g .

会触发：

```text
Direct publishing is not allowed
```

这是包里的保护逻辑。  
正确方式是：

```bash
sudo npm install -g @anthropic-ai/claude-code@版本号
```

或者使用镜像源安装。

---

## 4. 删除当前目录后要立刻 cd ~

如果你在这个目录里：

```bash
/usr/lib/node_modules/@anthropic-ai/claude-code
```

然后执行：

```bash
sudo rm -rf /usr/lib/node_modules/@anthropic-ai/claude-code
```

当前 shell 的工作目录就没了。  
再运行 npm 会报：

```text
uv_cwd
```

解决：

```bash
cd ~
```

---

## 5. DeepSeek 模型名先用 deepseek-chat

虽然目标是接入 DeepSeek V4 Flash，但在 Claude Code 环境变量里，直接写：

```bash
deepseek-v4-flash
```

可能报模型不存在。

更稳的写法：

```bash
export ANTHROPIC_MODEL="deepseek-chat"
```

如果后续确认 DeepSeek 兼容接口支持具体的 V4 Flash 模型名，再替换。

---

# 六、当前最终可用状态

当前板子已经完成：

1. USB 网络共享可用。
2. SSH 可用。
3. Node.js 可用。
4. npm 可用。
5. 旧版崩溃 Claude Code 已清理。
6. 正确路线是 npm 安装 Claude Code。
7. DeepSeek 环境变量写入 `~/.bashrc`。
8. 后续可直接运行 `claude` 作为终端 Agent 管理 RK3566 板子。

最终目标是让 Claude Code / DeepSeek 作为板子的运维 Agent，可以执行：

```text
检查系统负载
检查网络状态
检查 SSH 状态
分析 dmesg
分析 /dev/fb0
生成 Wi-Fi 自动重连脚本
生成 systemd 服务
辅助调试摄像头、eDP、Wi-Fi、USB 网络共享
```

这次折腾的核心结论：

在泰山派 RK3566 Ubuntu 22.04 上，Claude Code 不要用直接下载的 ARM64 二进制包，也不要用 `--ignore-scripts` 离线硬装。最稳路线是：先装 Node.js，再用 npm 镜像正规安装指定版本，然后用环境变量接入 DeepSeek。
```