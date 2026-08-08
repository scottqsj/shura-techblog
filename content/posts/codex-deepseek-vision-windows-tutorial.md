---
title: "Codex 接入 DeepSeek 教程｜ChatGPT 桌面端一键配置 + 识图技能"
date: 2026-08-08T00:00:00+08:00
draft: false
categories: ["AI知识技能", "脚本与工具教程"]
tags: ["Codex","DeepSeek","ChatGPT","千问","Vision","AI编程","Windows"]
toc: true
---

## 前言

Codex 是 OpenAI 推出的 AI 编程助手,官方订阅要花不少钱。好消息是 DeepSeek API 原生兼容 Codex 使用的 Responses 协议,官方提供了一键配置脚本,把模型提供方从 OpenAI 换成 DeepSeek,就能用 DeepSeek 的模型跑 Codex,成本和速度都友好得多。

本文记录完整实操流程:ChatGPT 桌面端(内置 Codex)→ 官方一键脚本接入 DeepSeek → 再让 Codex 自己安装 Claude Vision Skill 实现识图(图片识别走千问 API)。全程不需要手动改任何配置文件。

## 整体流程

只需四步:

1. 安装 ChatGPT 桌面端(内置 Codex)并运行一次(生成 `~/.codex` 配置目录)
2. 在 PowerShell 里执行 DeepSeek 官方一键脚本,选模型、填 API Key
3. 重启客户端验证生效
4. 在 Codex 对话框里让它去 GitHub 找 Claude Vision Skill 安装,提供千问 API Key 完成识图

整体架构如下图所示——三个客户端形态共用同一份配置文件,推理走 DeepSeek,识图走千问:

![Codex + DeepSeek + 千问 Vision 架构](https://img.shashura1.top/d/115pan/blog-img/codex-arch-v3.png)

## 准备工作

- Windows 10/11 电脑,准备安装 ChatGPT 桌面端
- DeepSeek API Key(以 `sk-` 开头,在 DeepSeek Platform 获取)
- 千问 API Key(阿里云百炼/DashScope 获取,用于识图;识别模型用 `qwen-vl-ocr-latest`)

提示:目前仅 `deepseek-v4-flash` 支持接入 Codex,`deepseek-v4-pro` 预计 2026 年 8 月初支持。另外 Codex CLI、ChatGPT 桌面端、VS Code 的 Codex 插件共用同一份配置文件,按本文配置一次,三种形态都能用 DeepSeek 模型。

## 第一步:安装 ChatGPT 桌面端

现在的 ChatGPT 桌面端已经把 Codex 功能合并进来了,不再需要单独安装 Codex CLI 或旧版 Codex 桌面应用——装一个 ChatGPT 客户端,Codex 就在里面。

### 1.1 下载安装包

浏览器打开 ChatGPT 官方下载页 <https://chatgpt.com/download>,选择 Windows 版本下载安装包(`ChatGPT_Setup.exe`)。

> 如果下载页访问慢或打不开,通常是网络问题,可以换网络环境或使用代理后再试。

### 1.2 安装

双击安装包,按提示完成安装。安装过程中:

- 默认安装到当前用户目录,无需管理员权限
- 安装完成后会自动创建桌面快捷方式「ChatGPT」

### 1.3 首次登录

打开 ChatGPT 桌面端,用 OpenAI 账号登录。登录成功后:

- 左侧边栏可以看到「Codex」入口(Codex 已内置)
- 先不用管模型选择,保持默认即可
- **至少让客户端运行一次再退出**——这一步很关键,它会生成 `~/.codex` 配置目录,DeepSeek 一键脚本要求这个目录已存在

> 小提示:账号没有 ChatGPT Plus 订阅也没关系,我们后面会用 DeepSeek 的 API 替代,配置好后不需要订阅也能用。

## 第二步:一键脚本接入 DeepSeek

Windows 用户打开 PowerShell(管理员或普通用户均可),执行:

```powershell
irm https://cdn.deepseek.com/api-docs/codex-deepseek-setup-en.ps1 | iex
```

> macOS / Linux 用户在终端执行:
> ```bash
> bash <(curl -fsSL https://cdn.deepseek.com/api-docs/codex-deepseek-setup.sh)
> ```

这条命令来自 DeepSeek 官方文档「接入 Codex」页面,下图是官方文档截图:

![DeepSeek 官方文档:接入 Codex](https://img.shashura1.top/d/115pan/blog-img/codex-deepseek-official-doc.png)

运行后按菜单选择要使用的模型。首次运行会提示输入 API Key(`sk-` 开头)。

脚本会自动完成以下工作,全程无需手动编辑文件:

1. **备份现有配置**:将 `~/.codex/config.toml` 备份到 `~/.codex/backup-deepseek/`,随时可以还原
2. **写入模型目录** `~/.codex/models.json`:向 Codex 声明 DeepSeek 模型的元数据(上下文窗口、推理强度档位、工具调用格式等),让它像用内置模型一样用 DeepSeek
3. **修改** `~/.codex/config.toml`:只改写必要字段,新增 `[model_providers.deepseek]` 配置段;你原有的 MCP 服务器、项目信任级别等配置全部保留。若有冲突字段会删除并逐条打印原因
4. **校验**:写入前校验语法,不合法就中止,不碰任何文件

想换模型或还原官方配置?再次运行脚本即可,菜单里有切换模型和恢复默认配置(第 3 项)的选项。

## 第三步:验证生效

重新打开 ChatGPT 桌面端(切换后需重启客户端才生效),模型选择器中:

- Windows 端可能显示「自定义」或「DeepSeek-V4-Flash」
- 显示为「自定义」时,实际使用的就是你选择的 DeepSeek 模型

在 Codex 对话框随便问一句,能正常回答就是配置成功了。

## 第四步:让 Codex 自己装识图技能

DeepSeek 模型本身不支持图片识别,但 Codex 可以装技能补上。这一步不需要自己找仓库、手动 clone,直接在 Codex 对话框里说清楚需求即可:

1. 告诉 Codex:去 GitHub 上找一个 Claude Vision Skill 安装
2. 提供千问 API Key,让它用千问的视觉模型(`qwen-vl-ocr-latest`)来完成识图

Codex 会自己去 GitHub 找到对应的 vision skill 仓库,读 README,完成安装和配置。装好后,你直接把图片丢给 Codex,它就能通过千问 API 识别图片内容了。

> 小提示:给 Codex 下指令时把需求说完整——"去 GitHub 找 Claude Vision Skill,用千问 API 做图片识别",它就知道该装哪个、怎么配了。

## 常见问题

**切到 DeepSeek 后历史会话不见了?**

没丢。Codex 按登录方式分组存放会话:ChatGPT 官方订阅的会话和第三方 API(DeepSeek)的会话分属两组,界面上只显示与当前配置匹配的一组。用脚本菜单第 3 项恢复原配置,官方会话就回来了;切回 DeepSeek 时它的会话也会重新出现。

**切换模型/提供方后没生效?**

重启 ChatGPT 客户端。配置文件改动需要重启才会加载。

**装坏了想还原?**

一键脚本每次运行前都会备份,`~/.codex/backup-deepseek/` 里就是原始配置,或者直接再跑一次脚本选恢复默认。

**VS Code 里能用吗?**

能。VS Code 的 Codex 插件与 Codex CLI / ChatGPT 桌面端共用同一份配置,装好插件直接就是 DeepSeek 模型。

## 总结

整个流程的核心就一条命令:

```powershell
irm https://cdn.deepseek.com/api-docs/codex-deepseek-setup-en.ps1 | iex
```

DeepSeek 官方把配置脚本化之后,接入门槛几乎为零:装好 ChatGPT 桌面端 → 跑脚本 → 填 Key,完事。识图能力则是通过让 Codex 安装 Claude Vision Skill + 千问 API 补齐,等于一套免费平替方案,编程、识图都能干。
