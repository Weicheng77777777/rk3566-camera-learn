# rk3566-camera-learn

记录学习RK3566camera

终于实现环境！
<img width="911" height="666" alt="image" src="https://github.com/user-attachments/assets/553cfce9-455a-4630-bc00-b9bc6f140efc" />

## linux配置Claudecode

```bash
cat >> ~/.bashrc << 'EOF'

# Claude Code 接入 DeepSeek 运营配置
# 使用 Anthropic 兼容接口地址
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
# 替换为你的真实 API Key
export ANTHROPIC_AUTH_TOKEN="替换为你的真实 API Key"
# 建议使用 deepseek-chat，它会自动指向最新的 V4 模型
export ANTHROPIC_MODEL="deepseek-v4-flash"
EOF

# 立即刷新环境
source ~/.bashrc
```

### 启动！

```bash
# 启动时开启 Auto mode
claude --enable-auto-mode
```

如果你是在 Docker、临时目录、虚拟机里跑，确认没有重要文件和敏感凭证，再用：
```bash
claude --dangerously-skip-permissions
```
