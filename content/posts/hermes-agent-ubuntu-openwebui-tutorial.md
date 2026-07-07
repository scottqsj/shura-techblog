---
title: "Hermes Agent 完全部署指南｜Ubuntu虚拟机 + 极空间NAS Open WebUI 联动教程"
date: 2026-07-07T00:00:00+08:00
draft: false
categories: ["NAS存储", "IT知识"]
tags: ["Hermes","AI Agent","Ubuntu","极空间","OpenWebUI","Docker"]
toc: true
---

## 前言

Hermes Agent 是 Nous Research 开源的 AI Agent 框架，支持记忆、技能、工具调用、定时任务、多平台接入等功能，可以接入任何 LLM 提供商（DeepSeek、OpenAI、Anthropic 等）。配合 Open WebUI 前端界面，你可以在任何设备上通过浏览器与 Hermes 交互，获得完整的 Agent 能力。

本文完整记录在 Ubuntu 虚拟机内安装 Hermes Agent + Desktop，然后在极空间 NAS 上通过 Docker 部署 Open WebUI 对接 Hermes 的全过程，附带全部可复制命令和踩坑排错。

## 一、整体架构说明

<table>
<tr>
  <th style="text-align:center;padding:8px 16px;background:var(--code-bg);border-radius:6px">Ubuntu 虚拟机 (内网)</th>
  <th style="text-align:center;padding:8px">← API 调用 →</th>
  <th style="text-align:center;padding:8px 16px;background:var(--code-bg);border-radius:6px">极空间 NAS (内网)</th>
</tr>
<tr>
  <td style="vertical-align:top;padding:8px 16px">
    Hermes Agent (CLI)<br>
    Hermes Desktop (GUI)<br>
    Hermes API Server<br>
    监听 0.0.0.0:8642
  </td>
  <td></td>
  <td style="vertical-align:top;padding:8px 16px">
    Open WebUI (Docker)<br>
    浏览器访问前端<br>
    http://NAS_IP:3000
  </td>
</tr>
</table>

核心思路：Ubuntu 虚拟机跑 Hermes Agent 并启动 API Server，极空间 Docker 跑 Open WebUI，后者通过内网调用 Hermes API Server 实现完整的 Agent 交互。

## 二、第一部分：Ubuntu 虚拟机安装 Hermes Agent

### 环境要求

Ubuntu 20.04 / 22.04 / 24.04，内存建议 4GB 以上。先更新系统：

```bash
sudo apt update && sudo apt upgrade -y
```

Hermes 依赖 Python 3.10+ 和 uv 包管理器。安装脚本会自动处理：

```bash
# 一键安装 Hermes（包含 uv + Python 依赖 + 启动器）
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装完成后，重新打开终端或执行：

```bash
source ~/.bashrc
```

验证安装：

```bash
hermes --version
```

输出类似 `hermes-agent x.x.x` 即表示安装成功。

### 首次配置

运行配置向导，选择模型和提供商：

```bash
hermes setup
```

按提示依次选择 Provider（例如 DeepSeek）和模型，填入 API Key。

也可以直接编辑配置文件：

```bash
hermes config edit
```

关键配置项示例（以 DeepSeek 为例）：

```yaml
model:
  default: deepseek-v3
  provider: deepseek
```

API Key 存放在 `.env` 文件：

```bash
hermes config env-path
# 输出路径，编辑该文件填入
# DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

配置完成后测试聊天：

```bash
hermes chat -q "你好，介绍一下你自己"
```

如果能正常回复，Hermes Agent 核心部署完成。

## 三、第二部分：启动 Hermes API Server

Hermes 内置了一个 OpenAI 兼容的 API Server，支持 Open WebUI、LobeChat、LibreChat 等所有兼容 OpenAI API 格式的前端连接。

### 3.1 配置并启动 API Server

编辑配置文件，启用 API Server 并设置端口和密钥：

```bash
hermes config edit
```

在 `platforms` 下添加（如果已存在则修改）：

```yaml
platforms:
  api_server:
    enabled: true
    port: 8642
    host: "0.0.0.0"
    extra:
      key: "你自定义的API密钥"
```

host 设为 0.0.0.0 表示监听所有网卡，允许局域网其他设备访问。

或者直接通过环境变量快速启用：

在 `~/.hermes/.env` 中添加：

```ini
API_SERVER_ENABLED=true
API_SERVER_PORT=8642
API_SERVER_HOST=0.0.0.0
API_SERVER_KEY=你自定义的API密钥
```

### 3.2 启动 Gateway（包含 API Server）

Hermes 的 API Server 作为 Gateway 的一部分运行：

```bash
# 前台运行（调试用）
hermes gateway run

# 后台服务运行（生产推荐）
hermes gateway install
hermes gateway start
```

### 3.3 验证 API Server

测试健康检查：

```bash
curl http://localhost:8642/health
```

返回 `{"status":"ok"}` 即表示 API Server 正常运行。

测试模型列表（OpenAI 兼容端点）：

```bash
curl -H "Authorization: Bearer 你自定义的API密钥" http://192.168.xx.xx:8642/v1/models
```

应返回包含 `hermes-agent` 模型的 JSON 列表。

## 四、第三部分：安装 Hermes Desktop（GUI 桌面端）

Hermes 提供原生 Electron 桌面应用，支持 macOS / Linux / Windows。

### 4.1 启动桌面端

如果是在带图形界面的 Ubuntu 虚拟机上：

```bash
hermes desktop
# 或者
hermes gui
```

首次启动会自动下载 Electron 运行环境。启动后可以看到完整的 GUI 界面：聊天窗口、会话列表、模型选择器、快捷键面板等。

### 4.2 无头虚拟机使用方案

如果虚拟机没有桌面环境，Hermes Desktop 无法直接运行。两种替代方案：

方案一：安装轻量桌面环境

```bash
sudo apt install xfce4 xfce4-goodies -y
# 安装后 startx 启动桌面，再运行 hermes desktop
```

方案二：远程访问（推荐）

在虚拟机内安装 xrdp 实现远程桌面：

```bash
sudo apt install xrdp xfce4 -y
sudo systemctl enable xrdp
sudo systemctl start xrdp
```

然后从 Windows / Mac 使用远程桌面客户端连接 `虚拟机IP:3389`，登录后在桌面终端运行 `hermes desktop`。

### 4.3 Desktop 连接远程 Gateway

Desktop 支持连接远程 Hermes Gateway（例如另一台服务器），这是 Headless 部署的优雅方案：虚拟机只跑 Gateway，本地电脑运行 Desktop 远程连接。

Desktop 启动后在设置中填入 Gateway 地址和认证信息即可。

## 五、第四部分：极空间 NAS 安装 Open WebUI

### 5.1 环境确认

极空间 NAS 支持 Docker，在 App 中心安装 Docker 应用后即可使用。

验证 Docker 可用：

```bash
docker --version
```

### 5.2 拉取并运行 Open WebUI

SSH 登录极空间 NAS，执行：

```bash
# 拉取官方镜像
docker pull ghcr.io/open-webui/open-webui:main

# 运行容器
docker run -d \
  --name open-webui \
  --restart always \
  -p 3000:8080 \
  -v /volume1/docker/open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

参数说明：

| 参数 | 含义 |
|------|------|
| `-d` | 后台运行 |
| `--name open-webui` | 容器名称 |
| `--restart always` | 开机自启 |
| `-p 3000:8080` | 端口映射，NAS 3000 端口映射容器 8080 |
| `-v /volume1/docker/open-webui:/app/backend/data` | 数据持久化目录 |

等待容器启动完成后，浏览器访问 `http://NAS内网IP:3000`，看到 Open WebUI 注册/登录界面即部署成功。

### 5.3 首次设置

1. 浏览器打开 `http://NAS内网IP:3000`
2. 创建管理员账号（首次注册即为管理员）
3. 登录后进入设置页面

## 六、第五部分：Open WebUI 对接 Hermes API Server

### 6.1 添加 Hermes 作为 OpenAI 兼容接口

进入 Open WebUI 管理后台：

左侧菜单 → 「管理员设置」→「外部连接」→「OpenAI API」

填写以下参数：

| 配置项 | 填写值 |
|--------|--------|
| 启用 | 打开开关 |
| API Base URL | `http://192.168.xx.xx:8642/v1`（替换为 Hermes 虚拟机的内网 IP） |
| API Key | `你自定义的API密钥`（与 API Server 配置的 key 一致） |
| 模型 ID | `hermes-agent` |

**关键点：模型名称必须填 `hermes-agent`**，这是 Hermes API Server 注册的唯一模型标识。

保存后，刷新页面。

### 6.2 选择 Hermes 模型对话

在 Open WebUI 顶部模型选择器中，应该能看到 `hermes-agent`。选中它即可开始与 Hermes Agent 对话。

### 6.3 验证完整 Agent 能力

在 Open WebUI 中测试 Hermes 的工具调用能力。例如发送：

> 帮我在 /tmp 目录下创建一个 test.txt 文件，写入当前时间

如果 Hermes 回复了创建文件的结果，说明工具调用链路畅通。你还可以测试：

- 文件读写操作
- 网络搜索
- Shell 命令执行
- 图片分析（如果配置了视觉模型）

## 七、踩坑记录

### 7.1 API Server 启动后内网无法访问

问题：虚拟机本地 `curl localhost:8642/health` 正常，但 NAS 访问 `http://虚拟机IP:8642` 超时。

原因：Ubuntu 默认防火墙 ufw 拦截了 8642 端口。

解决：

```bash
sudo ufw allow 8642/tcp
sudo ufw reload
```

如果使用云虚拟机或 Hyper-V，另外检查虚拟化平台的安全组/防火墙规则。

### 7.2 Open WebUI 连接返回 401 Unauthorized

问题：Open WebUI 页面提示连接失败，终端查看 Hermes 日志显示 401。

原因一：API Key 不匹配。检查 Open WebUI 中填写的 API Key 是否与 Hermes `.env` 中 `API_SERVER_KEY` 完全一致，注意不要有多余空格。

原因二：Open WebUI 请求没有携带 Authorization Header。确认 Open WebUI 设置中 API Key 不为空。

验证命令：

```bash
curl -H "Authorization: Bearer 你的密钥" http://虚拟机IP:8642/v1/models
```

如果返回正常，说明 API Server 本身没问题，排查 Open WebUI 配置。

### 7.3 Hermes API 返回 404 或模型不可用

问题：Open WebUI 能看到模型但对话报错 `model not found`。

原因：模型名称填写错误。

解决：Open WebUI 的模型 ID 必须填 `hermes-agent`（不是 deepseek-v3 或其他 Provider 实际的模型名）。Hermes API Server 统一对外暴露 `hermes-agent` 这个模型标识，内部由 Hermes 路由到配置的 Provider 模型。

### 7.4 Gateway 后台运行一段时间后停止

问题：`hermes gateway install` 安装的服务运行一段时间后自动退出。

原因：systemd user 服务在用户登出 SSH 后会被终止。

解决：

```bash
# 启用 linger 使 user 服务在登出后继续运行
sudo loginctl enable-linger $USER

# 重启 Gateway 服务
hermes gateway restart

# 检查状态
hermes gateway status
```

### 7.5 Desktop 启动报错：缺少图形依赖

问题：运行 `hermes desktop` 提示 `cannot open display` 或缺少 libgtk 等错误。

解决：安装必要的图形依赖库：

```bash
sudo apt install libgtk-3-0 libnotify4 libnss3 libxss1 libxtst6 xdg-utils libatspi2.0-0 libdrm2 libgbm1 libasound2 -y
```

如果虚拟机完全没有桌面环境，需要使用 xrdp（见 4.2 节）或改为在宿主机直接安装 Hermes Desktop 远程连接虚拟机 Gateway。

### 7.6 极空间 Docker 拉取镜像失败

问题：`docker pull ghcr.io/open-webui/open-webui:main` 超时或报 TLS 错误。

原因：ghcr.io 在国内访问不稳定。

解决：换用国内镜像源或手动导入镜像。

方式一：配置 Docker 镜像加速器（如果极空间支持）

方式二：在能正常拉取的机器上下载镜像后导出，再导入到极空间：

```bash
# 在能访问的机器上
docker pull ghcr.io/open-webui/open-webui:main
docker save ghcr.io/open-webui/open-webui:main -o open-webui.tar

# 传输 tar 到极空间后导入
docker load -i open-webui.tar
```

### 7.7 Open WebUI 对话能收到回复但没有工具调用

问题：对话正常，但 Hermes 无法执行文件操作、终端命令等。

原因：Hermes 的 API Server 支持下限工具集，默认已包含核心工具。如果缺失，检查 Hermes 工具配置：

```bash
hermes tools list
```

确认 `terminal`、`file`、`web` 等工具集处于启用状态。如果需要调整：

```bash
hermes tools enable terminal
hermes tools enable file
```

修改后重启 Gateway 生效：

```bash
hermes gateway restart
```

## 八、进阶用法

### 8.1 固定分配 API Key（多人使用）

Hermes 支持通过 `API_SERVER_KEY` 进行简单的 API 鉴权。如果需要多人多 Key，可以使用 Gateway 的 Session 机制，或者在前端 Open WebUI 侧通过用户权限控制。

### 8.2 外网访问（Cloudflare Tunnel）

如果想从外网访问 Open WebUI，参考前一篇 Hugo 博客教程中的 Cloudflare Zero Trust 隧道方案，将 NAS 的 3000 端口通过隧道暴露到公网域名。

### 8.3 多模型切换

Hermes 支持随时切换 Provider 模型。在 Open WebUI 中无需修改配置，只需在 `hermes config edit` 中修改 `model.default` 字段，重启 Gateway 即生效。也可以通过 `hermes model` 交互式切换。

### 8.4 定时任务

Hermes 内置 cron 调度系统，可以定期执行任务：

```bash
# 每天早上 9 点执行股票分析
hermes cron create "0 9 * * *" --prompt "分析今天的市场行情，发送到 Telegram"
```

## 九、日常使用流程

1. 确保 Hermes Gateway 服务运行中：`hermes gateway status`
2. 浏览器打开 Open WebUI：`http://NAS_IP:3000`
3. 顶部模型选择器切换到 `hermes-agent`
4. 正常对话，Hermes 将自动使用工具、记忆、技能完成复杂任务

虚拟机重启后需手动确认 Gateway 已启动：

```bash
hermes gateway status
# 如果未运行则
hermes gateway start
```

也可以将自启命令加入 crontab 或 systemd。

## 十、方案总结

这套方案的优势：

1. 完全本地部署，数据不外传（API 调用除外）
2. Hermes Agent 提供完整的工具链：文件操作、终端命令、网络搜索、子代理任务委派、持久记忆、技能学习
3. Open WebUI 提供跨设备浏览器访问的统一界面
4. 极空间 NAS 常年运行，Open WebUI 随时可用
5. 全部开源免费，无月租成本

核心链路：Hermes API Server（虚拟机）← 内网调用 → Open WebUI（NAS Docker）← 浏览器访问 → 任意设备。

后续如需扩展，可以接入 Telegram / Discord / Slack 等消息平台，实现随时随地通过聊天工具与 Hermes 交互。
