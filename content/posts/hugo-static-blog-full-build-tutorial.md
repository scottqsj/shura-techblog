
---
title: "从零搭建 Hugo 静态博客｜OpenList+网盘图床 + Cloudflare Pages 免费部署 全套命令&踩坑避坑"
date: 2026-07-04T00:00:00+08:00
draft: true
categories: ["NAS存储", "IT知识"]
tags: ["Hugo","静态博客","Cloudflare","OpenList","PicGo","ZeroTrust隧道"]
toc: true
---

## 前言
折腾两天完整落地一套免费静态博客方案：本地Markdown写作 + PicGo一键传图到私人网盘图床 + Gitee/GitHub双仓库同步 + Cloudflare Pages自动构建部署 + 自有域名全站HTTPS。
网上大多只有单独Hugo、单独Alist教程，极少有OpenList兼容隧道图床完整流程，本文附上全部可复制Linux命令，同时记录我踩过的所有报错坑，新手可直接照搬操作。

## 一、整套架构组件说明
1. Hugo：本地生成静态网页，写Markdown源文件
2. Git：Gitee为主仓库，单向同步GitHub备用
3. Cloudflare Pages：免费静态站点托管，博客主域名`blog.example.top`、图床二级域名`img.blog.example.top`
4. OpenList：虚拟机部署，挂载私人网盘充当博客图床，兼容Alist V3 API
5. Cloudflare Zero Trust 永久隧道：内网OpenList暴露公网二级图床域名
6. PicGo 3.0：截图一键上传图床，自动生成Markdown图片链接

## 二、第一部分：Linux 安装 Hugo & 初始化博客站点
### 环境：Ubuntu/Debian 虚拟机（NAS内置虚拟机通用）
#### 1. 系统更新依赖
```bash
sudo apt update && sudo apt upgrade -y
```

2. 安装 Git、解压工具
```bash
sudo apt install git unzip wget -y
```
3. 一键安装 Hugo（扩展版，支持图片 / 资源处理）
以 v0.146.0 版本举例，可自行替换最新版本号：
```bash
# 下载64位Linux二进制包
wget https://github.com/gohugoio/hugo/releases/download/v0.146.0/hugo_0.146.0_linux_amd64.tar.gz
# 解压
tar -zxvf hugo_0.146.0_linux_amd64.tar.gz
# 移动到系统全局命令目录
sudo mv hugo /usr/local/bin/
# 验证安装
hugo version
```
输出版本号即安装成功。

4. 初始化博客站点
```bash
# 创建博客文件夹
hugo new site ~/tech-blog
# 进入博客根目录
cd ~/tech-blog
```
5. 安装博客主题
```bash
# 拉取主题到themes目录，替换为你选用的主题仓库地址
git clone https://github.com/theme-author/hugo-theme-demo.git themes/demo-theme
```
6. 基础站点配置 config.toml
编辑配置文件：
```bash
nano config.toml
```
粘贴基础配置模板：
```toml
baseURL = "https://blog.example.top/"
languageCode = "zh-cn"
title = "个人技术笔记"
theme = "demo-theme"

# 开启草稿本地预览
buildDrafts = true
```
保存退出快捷键：Ctrl+O → 回车确认 → Ctrl+X

7. 新建第一篇测试文章
```bash
hugo new content/posts/test-first-article.md
```
打开文件编辑：
```bash
nano content/posts/test-first-article.md
```
将头部 draft = true 修改为 draft = false，代表正式发布文章。
8. 本地启动预览博客
```bash
# 允许局域网其他设备访问，本机1313端口打开站点
hugo server --bind 0.0.0.0
```
浏览器内网访问地址：http://192.168.xx.xx:1313 查看本地博客效果。
## 三、第二部分：Gitee 仓库初始化 + 单向同步 GitHub

### 3.1 本地配置 Git 账号（仅首次执行）

```bash
git config --global user.name "自定义Git昵称"
git config --global user.email "your-email@example.com"
```

### 3.2 生成 SSH 密钥（免密推送 Gitee/GitHub）

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

全程回车，无需设置密码。查看并复制公钥内容：

```bash
cat ~/.ssh/id_ed25519.pub
```

分别进入 Gitee、GitHub 个人设置 - SSH 公钥页面，粘贴保存密钥。

### 3.3 初始化本地 Git 仓库，关联 Gitee 主仓库

网页端 Gitee 新建空仓库 tech-blog，不要勾选初始化 README。回到博客目录执行：

```bash
cd ~/tech-blog
git init
git remote add gitee git@gitee.com:gitee-name/tech-blog.git
git add .
git commit -m "初始化博客源码仓库"
git push -u gitee main
```

### 3.4 单向同步方案：Gitee 更新自动同步 GitHub

方案 A：本地双远程手动推送（新手简单版）

```bash
git remote add github git@github.com:github-name/tech-blog.git
```

每次写完文章提交后，一键推送到两个平台：

```bash
git add .
git commit -m "更新文章：xxx"
git push gitee main
git push github main
```

方案 B：Gitee 网页自动镜像同步（推荐，无需双推）

Gitee 打开仓库 → 顶部「管理」→ 左侧「同步设置」→「仓库镜像同步」。镜像源填写 GitHub 仓库 SSH 地址，填入 GitHub SSH 公钥完成授权。开启 Gitee→GitHub 单向自动同步，勾选推送时同步，保存配置。后续仅需推送 Gitee，GitHub 会自动同步全部代码。

### 3.5 Gitee Pages 测试访问（可选）

Gitee 仓库 → 服务 → Gitee Pages，选择 main 分支、public 目录，开启后获得 `your-name.gitee.io/tech-blog` 测试地址，确认静态页面可正常访问。

### 3.6 GitHub 侧操作（可选）

进入 GitHub 仓库 → Settings → Pages，确认 Source 为 main 分支、/public 目录，获得 `your-name.github.io/tech-blog` 备用地址。实际生产由 Cloudflare Pages 接管，GitHub Pages 仅作备份验证。
## 四、第三部分：Cloudflare Pages 绑定仓库自动部署博客

### 4.1 进入 Cloudflare Pages 创建项目

登录 Cloudflare 后台，左侧菜单「Workers 和 Pages」→「Pages」→「创建应用程序」→「连接到 Git」。首次使用需授权 GitHub 或 Gitee 账号，选择对应的博客仓库 tech-blog。

### 4.2 配置构建参数

在「构建设置」页面填写以下参数：

| 配置项 | 填写值 |
|--------|--------|
| Framework preset | Hugo |
| 构建命令 | `hugo` |
| 构建输出目录 | `public` |
| 生产分支 | `main` |

环境变量无需额外添加。点击「保存并部署」，Cloudflare 会自动拉取仓库源码、执行 `hugo` 构建命令、将生成的 `public/` 目录部署到 `*.pages.dev` 临时域名。

### 4.3 等待首次部署完成

首次部署约需 1-2 分钟，完成后 Cloudflare 会提供一个 `项目名.pages.dev` 的预览地址。浏览器打开该地址，确认博客页面正常加载。

### 4.4 配置自定义域名

进入该 Pages 项目 →「自定义域」→「设置自定义域」，输入你的博客域名（如 `blog.example.top`）。Cloudflare 会自动为该域名生成 DNS 记录并提示你去 DNS 面板确认。

### 4.5 DNS 添加 CNAME 记录

回到 Cloudflare 首页 → 选中该域名 →「DNS」→「记录」，确认已自动添加一条：

| 类型 | 名称 | 目标 |
|------|------|------|
| CNAME | `blog` | `项目名.pages.dev` |

如果未自动生成，手动添加此条 CNAME 记录，代理状态保持「已代理」（橙色云朵图标）。

### 4.6 等待 SSL 证书自动签发

DNS 生效后约 1-5 分钟，Cloudflare 自动为该自定义域名签发 SSL 证书。进入「SSL/TLS」→「概述」，确认加密模式为「完全」或「完全（严格）」。

### 4.7 最终验证

浏览器访问 `https://blog.example.top`，确认：

- 地址栏显示锁形图标（HTTPS 已生效）
- 首页、文章列表、单篇文章页面均正常加载
- 后续 `git push gitee main` 后约 1 分钟，线上站点自动同步更新
登录 Cloudflare 后台，左侧「Workers 和 Pages」→ 创建应用 → 连接 Git
授权 Gitee/GitHub 账号，选中博客仓库 tech-blog
构建配置参数：
生产分支：main
构建命令：hugo
构建输出目录：public
环境变量无需额外添加，直接下一步部署
部署完成后，进入 Pages 设置 → 自定义域
添加主域名 blog.example.top，Cloudflare 自动生成 DNS 解析记录，前往域名 DNS 面板确认解析生效；
SSL/TLS→概述，加密模式改为「完全」，全站强制 HTTPS。
## 五、第四部分：虚拟机部署 OpenList（网盘图床）全套命令
1. 下载 OpenList Linux amd64 程序
```bash
# 新建工作目录
mkdir ~/openlist && cd ~/openlist
# 下载最新版OpenList，替换对应发布地址
wget https://github.com/OpenListRepo/openlist-linux-amd64
# 赋予执行权限
chmod +x openlist-linux-amd64
```
2. 后台常驻运行 OpenList（监听全部网卡 0.0.0.0:58348）
```bash
# 后台运行，日志输出到文件持久保存
nohup ./openlist-linux-amd64 --bind 0.0.0.0:58348 > openlist.log 2>&1 &
```
内网访问 OpenList 后台：http://192.168.xx.xx:58348
存储→添加存储，选择对应网盘，扫码登录完成挂载；
网盘根目录新建文件夹 /storage/blog-img，专门存放博客配图；
设置→其他，复制生成「API 令牌」，后续 PicGo 插件鉴权使用；
设置→全局设置，关闭「签名所有」开关（解决外网 401 鉴权报错），保存后重启 OpenList。
3. 设置 OpenList 开机自启（systemd 系统服务）
创建服务配置文件：
```bash
sudo nano /etc/systemd/system/openlist.service
```
写入以下内容，两处用户名替换为你的虚拟机登录账号：
```ini
[Unit]
Description=OpenList Image Bed Service
After=network.target

[Service]
User=linux-user
WorkingDirectory=/home/linux-user/openlist
ExecStart=/home/linux-user/openlist/openlist-linux-amd64 --bind 0.0.0.0:58348
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
重载服务配置并启用开机自启：
```bash
sudo systemctl daemon-reload
sudo systemctl start openlist
sudo systemctl enable openlist
# 查看服务运行状态
systemctl status openlist
```
## 六、第五部分：Cloudflare Zero Trust 永久隧道（暴露内网图床公网访问）
1. 官方安装 cloudflared 服务（Debian/Ubuntu 通用）
```bash
# 导入Cloudflare GPG密钥
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null

# 添加软件源
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# 更新软件源并安装
sudo apt-get update && sudo apt-get install cloudflared
```
2. Zero Trust 面板创建隧道并安装连接器
Cloudflare 账号首页进入「Zero Trust（零信任）」→ 左侧「网络」→「隧道」→ 创建隧道
隧道自定义名称openlist-image-bed，系统选择 Debian 64 位
复制页面内「安装服务」专属命令（携带唯一 token），粘贴到虚拟机终端执行：
```bash
sudo cloudflared service install tunnel-secret-token-string
```
启动隧道后台服务：
```bash
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
# 实时查看隧道运行日志
journalctl -u cloudflared -f
```
3. 隧道路由网页端配置
隧道页面→已发布应用程序路由→添加路由
子域名：img，域名：example.top
路径输入框完全留空，不填写任何正则匹配规则
服务类型：HTTP，转发 URL 填写内网 OpenList 地址 http://192.168.xx.xx:58348
点击保存，等待 30 秒 CDN 缓存生效。
## 七、第六部分：PicGo 3.0 Alist 插件完整配置（一键上传配图）
图床参数填写对照表
打开 PicGo → 图床列表 → alist → 新建配置
| 配置项 | 填写内容 |
|--------|----------|
| 配置项 | 填写内容 |
| alist 版本 | 3 |
| alist 地址 | http://192.168.xx.xx:58348 |
| 上传路径 | /storage/blog-img |
| 用户 token | OpenList 后台复制的 API 令牌 |
| 用户名 / 密码 | 全部留空 |
| 访问域名 | https://img.blog.example.top（必须带 https://） |
其余输入框全部留空，保存配置并设为默认图床。
上传测试
拖拽图片到 PicGo 仪表盘，上传完成自动复制标准 Markdown 图片链接，格式示例：
![图片名称](https://img.blog.example.top/d/storage/blog-img/demo.jpg)
## 八、全套搭建踩坑完整记录（可直接对照排错）

### 8.1 临时 trycloudflare 隧道关闭终端后域名失效，图片报 1016 Origin DNS error

问题：临时命令运行隧道，关闭终端 / 重启虚拟机后临时域名直接回收，所有图片链接外网 404。

解决方案：放弃临时隧道，使用 Zero Trust 永久隧道绑定自有二级域名 `img.blog.example.top`，域名永久固定，重启机器不受影响。

### 8.2 公网访问图片报 502 Bad Gateway，内网 OpenList 可正常打开

原因：OpenList 仅监听局域网内网 IP，未监听本地回环 127.0.0.1，隧道默认回环转发失败。

两种解决方式：

1. 隧道路由转发地址直接改为内网 IP `192.168.xx.xx:58348`（最简，无需修改 OpenList 启动参数）
2. OpenList 启动命令添加 `--bind 0.0.0.0:58348`，监听本机全部网卡

### 8.3 图片链接 http 明文访问报 400 Bad Request

报错提示：`The plain HTTP request was sent to HTTPS port`。

解决：PicGo 图床「访问域名」完整填写 `https://img.blog.example.top`，所有新上传图片自动生成 HTTPS 链接。

### 8.4 外网打开图片 401 Unauthorized，提示 expire missing

原因：OpenList 默认开启「签名所有文件」，第三方 Alist 插件无法生成适配 OpenList 的时效签名参数，裸链接鉴权失败。

解决：OpenList 后台全局设置关闭「签名所有」开关，保存后重启 OpenList 服务。

### 8.5 隧道路由填写路径正则 `^/blog`，图片路径匹配不到返回 404

解决：路由路径输入框清空，不做任何路径拦截，匹配全站所有访问路径。

### 8.6 网盘删除测试图片后，PicGo 相册仍保留缓存记录

完整清理流程：

1. OpenList 网页端删除不需要的测试图片，释放网盘存储空间
2. PicGo 相册右键单张删除图片记录，或设置内一键清除全部相册缓存

### 8.7 找不到 Cloudflare 隧道功能入口

误区：在单域名详情页「网络」菜单内查找隧道。

正确入口：返回 Cloudflare 账号总首页，左侧全局菜单栏打开 Zero Trust（零信任），隧道功能在该页面内。
## 九、日常写作完整标准流程
截图，使用 PicGo 快捷键一键上传图片，自动复制 Markdown 图片链接
虚拟机新建文章：hugo new content/posts/文章标题.md
编辑 md 文章，粘贴图片链接，本地预览排版效果：hugo server --bind 0.0.0.0
本地排版无误后提交代码推送至 Gitee 仓库
```bash
cd ~/tech-blog
git add .
git commit -m "发布新文章：xxx"
git push gitee main
```
Gitee 自动同步 GitHub 仓库，Cloudflare Pages 自动构建，1 分钟后线上站点更新文章。
## 十、方案总结
整套方案全部依托免费服务搭建，无服务器月租成本，利用闲置私人网盘作为图床存储，Cloudflare 提供免费全球加速与 SSL HTTPS 证书。
搭建难点集中在 OpenList 与 Cloudflare 隧道的兼容适配，网上通用原版 Alist 教程无法直接套用，本文全部命令、配置、报错均为实战验证，后续重新搭建同类博客可直接复制操作。
