# Ollama + OpenClaw Docker 部署指南

在 Docker 中一键部署 Ollama 和 OpenClaw，实现本地 AI 助手运行。

## 📋 目录

- [前置要求](#前置要求)
- [快速开始](#快速开始)
- [详细安装步骤](#详细安装步骤)
- [配置说明](#配置说明)
- [常用命令](#常用命令)
- [故障排查](#故障排查)
- [FAQ](#faq)

---

## 前置要求

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| WSL2 | 最新版 | Windows Subsystem for Linux |
| Docker Desktop | 最新版 | 需启用 WSL2 后端 |
| Git Bash | 推荐 | Windows 下执行脚本 |

---

## 快速开始

```bash
# 1. 克隆 OpenClaw 仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 2. 执行安装脚本（如有代理需求）
http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 ./docker-setup.sh

# 3. 获取 Dashboard URL
clawdock-dashboard

# 4. 批准设备配对（如有需要）
clawdock-devices
clawdock-approve <request-id>
```

---

## 详细安装步骤

### 1. 安装 WSL2

下载并安装 [WSL2](https://learn.microsoft.com/zh-cn/windows/wsl/install)

### 2. 安装 Docker Desktop

- 下载 [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- 确保 Docker Desktop 正在运行
- 在设置中启用 WSL2 后端

### 3. 安装 OpenClaw

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 执行安装脚本（根据网络情况配置代理）
http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 ./docker-setup.sh
```

> **注意**：代理地址 `127.0.0.1:7890` 需替换为你实际的代理地址

### 4. 安装 ClawDock（命令行工具）

ClawDock 简化了 OpenClaw 的复杂操作。

#### 4.1 配置 OpenClaw 工程路径

在 `C:\Users\<用户名>` 下创建 `.bashrc` 文件
添加以下内容（替换为你的实际路径，一定要是全正斜杠/）

```bash
export CLAWDOCK_DIR="/c/Users/YOUR_USERNAME/openclaw"
```
使配置生效
```bash
source ~/.bashrc
```
验证配置
```bash
echo $CLAWDOCK_DIR
```

#### 4.2 下载 ClawDock

```bash
# 创建文件夹并下载脚本
mkdir -p ~/.clawdock && \
curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/shell-helpers/clawdock-helpers.sh \
  -o ~/.clawdock/clawdock-helpers.sh

# 添加到 shell 配置（zsh 或 bash，这里的示例是 zsh）
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
echo 'source ~/.zshrc' >> ~/.bash_profile
```

### 5. 安装 Ollama

（不推荐本地部署模型，费力不讨好。直接接提供商的 API 比较好）

```bash
# 拉取模型
docker exec ollama ollama pull qwen2.5:7b

# 验证 GPU 挂载
docker exec ollama nvidia-smi

# 运行模型测试
docker exec -it ollama ollama run qwen2.5:7b
```

---

## 配置说明

### 解决 Control UI 启动错误

如遇到以下错误：

```
Gateway failed to start: Error: non-loopback Control UI requires gateway.controlUi.allowedOrigins
```

在 `C:\Users\你的用户名\.openclaw\openclaw.json` 的 `gateway` 节点添加：

```json
{
  "gateway": {
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789",
        "http://192.168.x.x:18789"
      ]
    }
  }
}
```

### 配置工具权限

OpenClaw 默认权限为 messaging，如需完整权限，在 `openclaw.json` 根节点添加：

```json
{
  "tools": {
    "profile": "full"
  }
}
```

### 配置 Ollama 外部网络

在 OpenClaw 的 `docker-compose.yml` 中添加：

```yaml
networks:
  default:
  ollama_network:
    external: true
    name: ollama-openwebui_default  # 根据你的 docker-compose 文件名调整
```

### 设置本地 Ollama 模型

```bash
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

---

## 常用命令

### 设备配对

```bash
# 查看配对请求
clawdock-devices

# 批准配对
clawdock-approve <request-id>

# 获取 Dashboard URL
clawdock-dashboard
```

### Ollama 模型管理

```bash
# 查看 GPU 使用情况
docker exec -it ollama watch -n1 nvidia-smi

# 停止模型
docker exec -it ollama ollama stop qwen2.5:7b

# 重启模型
docker exec -it ollama ollama run qwen2.5:7b
```

### 文件权限修复

```bash
# 进入容器
docker exec -it openclaw-openclaw-gateway-1 /bin/bash

# 修复权限
chown -R node:node /home/node/.openclaw/workspace

# 验证权限
ls -ld /home/node/.openclaw/workspace
```

---

## 故障排查

### 问题 1：配对请求 (pairing required)

```bash
# 1. 查看待处理的请求
clawdock-devices

# 2. 复制 Request 列的 ID
# 3. 批准请求
clawdock-approve <request-id>
```

---

## FAQ

**Q: openclaw Dashboard 无法访问？**  
A: 检查 `openclaw.json` 中的 `allowedOrigins` 配置，确保包含你访问的 URL。

**Q: openclaw 无法读写文件？**  
A: 按需求调整 `openclaw.json` 文件的 `tools.profile`，默认的 profile 是 messaging，无法读写文件。

```json
{
    "tools": {
        "profile": "full"
    }
}
```

**Q: 如何确认 GPU 挂载成功？**  
A: 运行 `docker exec ollama nvidia-smi`，能看到 GPU 信息即表示成功。

---

## 相关资源

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [社区 Discord](https://discord.com/invite/clawd)

---

*最后更新：2026-03-09*
