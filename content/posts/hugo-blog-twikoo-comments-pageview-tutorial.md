---
title: "Hugo 静态博客免费加装评论区与浏览量统计｜Twikoo、Cloudflare Workers、D1 全套实录（附避坑指南）"
date: 2026-07-18T00:00:00+08:00
draft: false
categories: ["IT知识", "脚本与工具教程"]
tags: ["Hugo", "Twikoo", "Cloudflare Workers", "D1", "静态博客", "评论系统", "浏览量统计"]
toc: true
cover:
  image: "https://img.shashura1.top/d/115pan/blog-img/hugo-comment-pv-cover.jpg"
  alt: "Hugo 博客评论区与浏览量统计"
  hiddenInList: true
---

## 前言

静态博客天生有两个"残疾"：没有评论区，没有浏览量。页面是纯 HTML，没有后端，读者看完想说句话都没地方留。

网上教程大多让你用 LeanCloud + Vercel 部署 Waline，我原本也打算这么干，结果一路撞墙：

1. LeanCloud 国际版已经**关闭新用户注册**，老教程全部作废
2. Vercel 新账号动不动卡在"requires further verification"，GitHub 登录也过不去
3. Waline 官方**根本不支持 Cloudflare Workers**，此路彻底不通

最后换 **Twikoo**，整套方案全部跑在 Cloudflare 上：评论后端是 CF Workers，数据存 CF D1 数据库，浏览量统计是自己写的 30 行 Worker，复用同一个数据库。**月成本 0 元，数据全在自己账号里**，不依赖任何第三方评论平台。

本文是完整实操记录，所有命令可直接照搬。我的博客本体搭建过程见上一篇《从零搭建 Hugo 静态博客》，本文假设你已经有一个跑在 Cloudflare Pages 上的 Hugo 博客。

## 一、选型：为什么是 Twikoo

先回答最多人关心的问题：**读者评论要不要注册账号？**

| 评论系统 | 读者要账号吗 | 后端部署 | 坑 |
|---------|------------|---------|-----|
| giscus / utterances | 要 GitHub 账号 | 免部署 | 国内读者大多没有 GitHub |
| Disqus | 要 Disqus 账号 | 免部署 | 国内直接打不开 |
| Waline | 不要，匿名可评 | Vercel 等 | LeanCloud 停注册、不支持 CF |
| **Twikoo** | **不要，昵称即可** | **CF Workers 官方支持** | 基本没坑 |

Twikoo 是国产开源项目，中文界面，读者**填个昵称就能评论**（邮箱选填且不公开），带管理面板、评论审核、邮件通知。官方提供 `twikoo-cloudflare` 仓库，直接部署到 Cloudflare Workers，数据存 D1（Cloudflare 的免费 SQLite 数据库，每天 500 万次读取额度，个人博客用不完）。

浏览量统计则没有现成的好方案：不蒜子是第三方服务经常抽风，Cloudflare Web Analytics 只能在后台看不能显示在页面上。索性自己写——**30 行代码的 Worker 计数器**，复用 Twikoo 那个 D1 数据库，零成本。

## 二、整体架构

![架构图](https://img.shashura1.top/d/115pan/blog-img/hugo-comment-pv-arch.png)

三个服务全部住在同一个 Cloudflare 免费账号里：

1. **博客站点**：Cloudflare Pages，Hugo 静态页面（已有）
2. **Twikoo 评论服务**：CF Worker，绑定二级域名 `twikoo.你的域名`
3. **浏览量计数器**：CF Worker，绑定二级域名 `pv.你的域名`
4. **D1 数据库**：评论表和浏览量表放一起，一个库通吃

读者浏览器打开文章后，页面里的 JS 分别请求两个 Worker：拉取/提交评论、给阅读数 +1。

## 三、准备工作

### 1. 本机环境

需要 Node.js 18 以上和 npm（用于运行 wrangler 部署工具）。国内直连 npm 官方源经常超时，统一走淘宝镜像：

```bash
node -v   # >= 18
npm install -g wrangler --registry=https://registry.npmmirror.com
```

### 2. 创建 D1 数据库

Cloudflare 后台左侧菜单 → **存储和数据库（Storage & Databases）→ D1** → 创建数据库，名字随意（我用的 `shura-waline`）。

> 注意：D1 不在 Workers 和 Pages 菜单里面，是独立的一栏，别找错地方。

创建后进入数据库详情页，记下顶部的 **Database ID**（一串 uuid）。

### 3. 创建 API Token

wrangler 部署需要凭证。到 [API 令牌页面](https://dash.cloudflare.com/profile/api-tokens) → 创建令牌 → 选 **"编辑 Cloudflare Workers"** 模板，然后**必须手动加一条权限**：

```text
+ 添加更多 → 帐户 → D1 → 编辑
```

模板默认不含 D1 权限，不加这条后面初始化数据库必失败。

两个坑：

1. 别选成 **"Workers AI"** 模板——那个只能调 AI 推理接口，部署不了 Worker 也碰不了 D1
2. Token 用完之后记得回后台删掉，不要长期留着

拿到 Token 先验证一下有没有效（养成好习惯，快速失败）：

```bash
curl -s "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer 你的TOKEN"
# 返回 "status":"active" 即有效

curl -s "https://api.cloudflare.com/client/v4/accounts" \
  -H "Authorization: Bearer 你的TOKEN"
# 能列出你的 Account ID 才说明模板选对了
```

## 四、部署 Twikoo 评论后端

### 1. 克隆官方仓库

GitHub 国内需要代理，二选一：

```bash
cd ~/proj

# 有代理
git -c http.proxy=socks5h://127.0.0.1:10808 clone --depth 1 \
  https://github.com/twikoojs/twikoo-cloudflare.git

# 无代理，走 gh-proxy 镜像
git clone --depth 1 https://gh-proxy.com/https://github.com/twikoojs/twikoo-cloudflare.git

cd twikoo-cloudflare
npm install --registry=https://registry.npmmirror.com
```

### 2. 掏空三个不兼容的包

Workers 运行时跑不了这三个 Node 专属包，而且不处理会超出免费版 1MiB 体积限制。官方 README 的做法是直接把入口文件清空：

```bash
echo "" > node_modules/jsdom/lib/api.js
echo "" > node_modules/tencentcloud-sdk-nodejs/tencentcloud/index.js
echo "" > node_modules/nodemailer/lib/nodemailer.js
```

### 3. 修改 wrangler.toml

改 `[[d1_databases]]` 段，填入自己的数据库信息：

```toml
[[d1_databases]]
binding = "DB"
database_name = "shura-waline"          # 你的库名
database_id = "68929ba3-xxxx-xxxx-xxxx" # 你的 Database ID
```

同时**删掉整个 `[[r2_buckets]]` 段和 `[vars]` 里的 R2_PUBLIC_URL 行**（那是评论传图用的，需要另开 R2 存储桶；不删的话指向不存在的桶，部署直接报错）。

### 4. 初始化表结构并部署

```bash
export CLOUDFLARE_API_TOKEN="你的TOKEN"
export CLOUDFLARE_ACCOUNT_ID="你的AccountID"

# 建表，成功会显示 num_tables: 3
npx wrangler d1 execute shura-waline --remote --file=./schema.sql -y

# 部署，成功会输出一个 *.workers.dev 地址
npx wrangler deploy --minify
```

### 5. 绑定自定义域名（国内访问的生死线）

**重点坑：`*.workers.dev` 域名在国内被墙，直连完全打不开。** 部署完直接把 workers.dev 地址填进博客，评论区必然加载失败，而且是静默失败，页面上什么都看不到。

解法：给 Worker 绑一个自己域名的二级域名。可以在后台点（Worker 详情 → 设置 → 域和路由 → 添加自定义域），也可以一条 API 搞定：

```bash
# 查 zone_id
curl -s "https://api.cloudflare.com/client/v4/zones" \
  -H "Authorization: Bearer 你的TOKEN" | grep -o '"id":"[^"]*"' | head -1

# 绑定 twikoo.你的域名
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/你的AccountID/workers/domains" \
  -H "Authorization: Bearer 你的TOKEN" -H "Content-Type: application/json" \
  -d '{"zone_id":"你的ZoneID","hostname":"twikoo.你的域名","service":"twikoo-cloudflare","environment":"production"}'
```

等一两分钟证书签发，然后验证（这条必须做）：

```bash
curl -s https://twikoo.你的域名/
# 返回 {"code":100,"message":"Twikoo 云函数运行正常","version":"1.6.44"} 即成功
# 记下这个 version 号，下一步要用
```

## 五、Hugo 前端接入评论区

### 1. hugo.toml 加配置

PaperMod 主题在 `[params]` 下加：

```toml
comments = true

[params.twikoo]
  envId = "https://twikoo.你的域名"   # 必须是自定义域名，别填 workers.dev
  version = "1.6.44"                  # 和上面健康检查返回的版本一致
```

`comments = true` 放在站点配置里就是全站文章生效，不需要每篇文章的 frontmatter 单独加。

### 2. 下载 Twikoo 前端 JS 到本站

前端 JS 我最初用公共 CDN，结果又踩一坑：jsdelivr 国内时好时坏，npmmirror 虽然快但我这里 Linux 下测试正常、Windows 浏览器却加载失败，玄学问题排查半天。最后干脆**把 JS 下载到博客里自托管**——博客本身在 CF Pages 上，只要博客能打开，评论区就必然能加载，一劳永逸：

```bash
cd 你的博客目录
mkdir -p static/js
curl -sL -o static/js/twikoo.all.min.js \
  "https://registry.npmmirror.com/twikoo/1.6.44/files/dist/twikoo.all.min.js"
# 验证下载完整，约 600KB
ls -la static/js/twikoo.all.min.js
```

### 3. 创建评论模板

新建 `layouts/partials/comments.html`（PaperMod 预留的钩子，检测到这个文件自动加载）：

```html
{{/* Twikoo 评论区 — 服务端: Cloudflare Workers + D1 */}}
{{ with .Site.Params.twikoo.envId }}
<div class="twikoo-container">
  <div id="tcomment"></div>
</div>
<script src="/js/twikoo.all.min.js"></script>
<script>
  twikoo.init({
    envId: {{ . }},
    el: '#tcomment',
    lang: 'zh-CN'
  });
</script>
{{ end }}
```

**这里有个隐蔽大坑**：`envId: {{ . }}` 千万不要画蛇添足写成 `{{ . | jsonify }}`。Hugo 在 script 标签内会自动做上下文转义，本身就会给字符串加引号；再加 jsonify 会变成双重引号，生成 `envId: "\"https://...\""` 这种坏 JS，Twikoo 静默失败，页面上毫无报错。

改完构建后一定要验证渲染产物：

```bash
hugo --gc
grep -A3 "twikoo.init" public/posts/任意文章/index.html
# 期望看到 envId:"https://twikoo.你的域名" 这样干净的字符串
```

### 4. 深色模式适配

PaperMod 切深色是给 body 加 `dark` 类，而 Twikoo 用的 Element-UI 组件默认白底，深色模式下会刺眼。在 `assets/css/extended/custom.css` 加：

```css
.twikoo-container {
    margin-top: 40px;
    padding-top: 30px;
    border-top: 1px solid var(--border);
}

body.dark #tcomment .el-input__inner,
body.dark #tcomment .el-textarea__inner {
    background: var(--entry);
    color: var(--primary);
    border-color: var(--border);
}

body.dark #tcomment .tk-content,
body.dark #tcomment .tk-nick,
body.dark #tcomment .tk-comments-title {
    color: var(--primary);
}

body.dark #tcomment .tk-action-icon,
body.dark #tcomment .tk-time {
    color: var(--secondary);
}
```

### 5. 上线并抢注管理员

构建推送上线后，**第一时间打开任意文章，点评论框右上角的齿轮图标，设置管理员密码**。Twikoo 的规则是先到先得，第一个注册的就是管理员——一个无主的评论系统管理面板，被路人抢注就尴尬了。

管理面板里可以配置评论审核、垃圾过滤、邮件通知等。

上线效果不用截图——本文底部就是活的评论区，滚到最下面即可体验：读者只需填个昵称，秒发评论，无需注册任何账号。

## 六、30 行代码自建浏览量统计

Twikoo 只管评论，不带浏览量。自己写一个 Worker，逻辑极简：收到请求 → 按文章路径在 D1 里 +1 → 返回计数。

### 1. 项目文件

新建目录 `pv-counter`，三个文件。

`schema.sql`——就一张表：

```sql
CREATE TABLE IF NOT EXISTS pageviews (
  path TEXT PRIMARY KEY,
  count INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT
);
```

`wrangler.toml`——注意直接在配置里声明自定义域名，部署时自动绑定，不用再调 API：

```toml
name = "pv-counter"
main = "worker.js"
compatibility_date = "2024-09-23"

routes = [
  { pattern = "pv.你的域名", custom_domain = true }
]

[[d1_databases]]
binding = "DB"
database_name = "shura-waline"          # 复用 Twikoo 的库
database_id = "68929ba3-xxxx-xxxx-xxxx"
```

`worker.js`——完整代码如下，带 CORS 白名单和路径清洗：

```javascript
// 迷你浏览量计数器：复用 D1 数据库，表 pageviews
// GET /?path=/posts/xxx/&hit=1  -> 计数 +1 并返回 {count}
// GET /?path=/posts/xxx/        -> 只查询返回 {count}

const ALLOWED_ORIGINS = new Set([
  "https://你的博客域名",
  "http://localhost:1313",
]);

export default {
  async fetch(request, env) {
    const origin = request.headers.get("Origin") || "";
    const cors = {
      "Access-Control-Allow-Origin": ALLOWED_ORIGINS.has(origin)
        ? origin
        : "https://你的博客域名",
      "Access-Control-Allow-Methods": "GET, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type",
      "Vary": "Origin",
    };

    if (request.method === "OPTIONS")
      return new Response(null, { status: 204, headers: cors });
    if (request.method !== "GET")
      return Response.json({ error: "method not allowed" }, { status: 405, headers: cors });

    const url = new URL(request.url);
    let path = url.searchParams.get("path") || "";
    try { path = decodeURIComponent(path); } catch (e) {}
    path = path.split("?")[0].split("#")[0];

    if (!path.startsWith("/") || path.length > 256)
      return Response.json({ error: "bad path" }, { status: 400, headers: cors });
    if (!path.endsWith("/")) path += "/";

    const hit = url.searchParams.get("hit") === "1";
    let count = 0;

    if (hit) {
      const row = await env.DB.prepare(
        "INSERT INTO pageviews (path, count, updated_at) VALUES (?1, 1, datetime('now')) " +
        "ON CONFLICT(path) DO UPDATE SET count = count + 1, updated_at = datetime('now') " +
        "RETURNING count"
      ).bind(path).first();
      count = row ? row.count : 1;
    } else {
      const row = await env.DB.prepare(
        "SELECT count FROM pageviews WHERE path = ?1"
      ).bind(path).first();
      count = row ? row.count : 0;
    }

    return Response.json({ path, count }, {
      headers: { ...cors, "Cache-Control": "no-store" },
    });
  },
};
```

### 2. 建表并部署

```bash
cd pv-counter
export CLOUDFLARE_API_TOKEN="你的TOKEN"
export CLOUDFLARE_ACCOUNT_ID="你的AccountID"

npx wrangler d1 execute shura-waline --remote --file=./schema.sql -y
npx wrangler deploy
```

部署完验证计数逻辑（连打两次 hit，数字应该递增；不带 hit 只查不涨）：

```bash
curl -s "https://pv.你的域名/?path=/posts/test/&hit=1"   # {"count":1}
curl -s "https://pv.你的域名/?path=/posts/test/&hit=1"   # {"count":2}
curl -s "https://pv.你的域名/?path=/posts/test/"          # {"count":2}
```

## 七、Hugo 接入浏览量显示

新建 `layouts/partials/extend_footer.html`（也是 PaperMod 的钩子）。逻辑：只在文章页执行，把「阅读 N 次」追加到标题下方的日期后面；用 sessionStorage 防止同一会话内刷新重复计数；请求失败就静默跳过，不影响读文章：

```html
{{/* 文章浏览量统计：调用 pv Worker，数据存自己的 D1 */}}
{{ if and (eq .Kind "page") (eq .Type "posts") }}
<script>
(function () {
  var meta = document.querySelector('.post-single .post-meta');
  if (!meta) return;
  var path = location.pathname;
  var key = 'pv:' + path;
  var hit = sessionStorage.getItem(key) ? '' : '&hit=1';
  fetch('https://pv.你的域名/?path=' + encodeURIComponent(path) + hit)
    .then(function (r) { return r.json(); })
    .then(function (d) {
      if (typeof d.count !== 'number') return;
      sessionStorage.setItem(key, '1');
      var span = document.createElement('span');
      span.textContent = '\u00A0\u00B7\u00A0阅读 ' + d.count + ' 次';
      meta.appendChild(span);
    })
    .catch(function () {});
})();
</script>
{{ end }}
```

构建推送后，每篇文章标题下方的日期旁边就会出现「阅读 N 次」——效果同样不用截图，抬头看本文标题下方就是。

## 八、踩坑总结

这一路踩的坑，按杀伤力排序：

| 坑 | 症状 | 解法 |
|----|------|------|
| workers.dev 被墙 | 评论区静默加载失败，页面无报错 | 必须绑自定义域名 |
| LeanCloud 停注册 | 网上 Waline 老教程全废 | 换 Twikoo 加 CF 全家桶 |
| Vercel 新号卡验证 | GitHub 登录也过不去 | 弃用，全走 Cloudflare |
| 公共 CDN 玄学 | 同一个 JS 有人能加载有人不能 | 下载到博客 static 目录自托管 |
| Hugo 模板转义 | envId 双重引号，Twikoo 静默失败 | script 内直接 `{{ . }}`，别加 jsonify |
| API Token 模板选错 | 能验证通过但列不出账户 | 用"编辑 Workers"模板并手动加 D1 权限 |
| R2 配置残留 | wrangler 部署报错 | 删掉 wrangler.toml 里 R2 相关段落 |
| 管理面板无主 | 可能被路人抢注管理员 | 上线后立刻用齿轮图标设密码 |

另外两条通用经验：

1. **每一步都当场验证**：curl 健康检查、grep 渲染产物、连打两次计数看递增。别等全部做完再测，出问题都不知道哪层挂的。
2. **凭证卫生**：API Token 用完就删；聊天记录、截图里出现过的 Token 一律视为泄露，及时轮换。

## 九、成本与总结

| 项目 | 服务 | 费用 |
|------|------|------|
| 评论后端 | Cloudflare Workers 免费版（每天 10 万次请求） | 0 元 |
| 浏览量后端 | 同上，自建 30 行 Worker | 0 元 |
| 数据库 | Cloudflare D1 免费版（每天 500 万次读取） | 0 元 |
| 前端 JS | 自托管在 CF Pages | 0 元 |
| **合计** | | **每月 0 元** |

至此，静态博客补齐了动态能力：读者不用注册任何账号就能评论，每篇文章有阅读计数，评论和浏览量数据全部存在自己 Cloudflare 账号的 D1 数据库里——不怕第三方评论服务跑路，也不用担心哪天又一个"LeanCloud 停止注册"把你打回原形。

配合上一篇的博客本体加图床，整套个人博客基础设施的月成本依然是 0 元。静态博客的省心和动态博客的互动，这次真的可以兼得。
