---
title: GitHub Pages 博客搭建指南（Hexo + Butterfly）
date: 2026-10-09 11:55:21
categories: [学习笔记]
tags: [算法, 数据结构]
---

GitHub Pages 个人博客搭建指南（Hexo + Butterfly）
> **目标**：在你的 GitHub 账号下用 `<你的用户名>.github.io` 跑起个人博客，用于整理学习资料、记录日志。
> **方案**：Hexo（静态站点生成器）+ Butterfly（主题）+ GitHub Actions 自动构建部署——本地只写 Markdown，`git push` 后网站自动更新。
> **适配环境**：macOS（zsh 终端），已确认本机具备 Node.js v24.7.0、npm 11.5.1、Git、gh CLI（已登录 GitHub）。
> **版本基线**（2026-10 时点）：Hexo 8.x、Butterfly 5.x、hexo-generator-searchdb 1.5.x。

---

## 0. 你将得到什么

- 一个公开博客站点：`https://<你的用户名>.github.io`
- 本地用 Markdown 写作，推送到 GitHub 后**自动**构建发布，无需手动上传 HTML
- 自带能力：本地全文搜索、文章分类/标签/归档、深色模式（Butterfly 内置）
- 后续可平滑升级：评论（giscus）、独立页面（关于/书单）、自定义域名（指南含说明）

---

## 1. 原理与整体流程

```
你的 Mac                              GitHub
┌────────────────┐    git push    ┌──────────────────────────────┐
│ Hexo 源文件      │ ─────────────▶ │ 仓库：<用户名>.github.io        │
│ (Markdown/配置) │               │   └─ GitHub Actions 自动构建    │
└────────────────┘               │        hexo generate           │
                                 │        └─ public/ 产物 → Pages  │
                                 └──────────────────────────────┘
```

两个关键认知：

1. 你推送的是**源文件**（Markdown、配置、主题设置），不是 HTML 产物。HTML 由 GitHub 的服务器在每次推送后自动生成。
2. 仓库名必须叫 `<你的用户名>.github.io`。这是 GitHub 的特例规则：该名字的仓库，其 Pages 直接发布在根域名 `https://<你的用户名>.github.io` 下，浏览器访问路径最简单。

---

## 2. 准备工作

### 2.1 复核环境

```bash
node -v            # 期望 v24.x
npm -v             # 期望 11.x
git --version
gh auth status     # 确认已登录 GitHub；若未登录执行 gh auth login
gh api user --jq .login   # 输出你的 GitHub 用户名，替换下文所有 <你的用户名>
```

### 2.2 关键：npm 源处理（你环境特有，必须先做）

你的 npm 当前指向公司内网源（`registry.anpm.alibaba-inc.com`）。内网源地址会被写进 `package-lock.json`，而 GitHub 的构建机器**访问不到**内网源，会直接构建失败。

**对策**：在博客项目根目录放一个项目级 `.npmrc`，把源指向公共镜像，并且**提交到仓库**，保证本地与 CI 行为一致：

```bash
# 在博客项目根目录执行
echo "registry=https://registry.npmmirror.com" > .npmrc
```

并确保依赖和 lock 文件是在该配置下生成的（第 3 步会做）。检查方法：

```bash
grep -c "anpm.alibaba-inc.com" package-lock.json   # 输出必须是 0
```

### 2.3 安装 Hexo CLI

```bash
npm install -g hexo-cli
hexo version       # 期望 4.3.x
```

> 若全局安装报权限错误（EACCES），可改用 `npx hexo <命令>` 的方式，或按 npm 官方建议重配全局目录前缀。

---

## 3. 第一步：本地创建 Hexo 站点

```bash
cd ~/workspace                  # 选一个放博客源码的目录（自行替换）
hexo init my-blog               # 生成站点骨架（会自动执行一次 npm install）
cd my-blog

# —— 立刻切到公共源并重建依赖，避免内网源污染 lock 文件 ——
echo "registry=https://registry.npmmirror.com" > .npmrc
rm -rf node_modules package-lock.json
npm install
```

本地预览：

```bash
hexo server                     # 或 npm run server
# 浏览器打开 http://localhost:4000，应看到默认的 Hello World 文章
```

说明：

- `hexo init` 生成的目录**自带合适的 `.gitignore`**（忽略 `node_modules/`、`public/`、`db.json`、`*.log` 等），无需自己改。注意 `public/` 是构建产物，永远不要提交。
- 站点基础信息在**根目录 `_config.yml`** 中修改：

```yaml
title: 我的学习笔记 # 站点标题
subtitle: ""
description: ""
author: <你的名字>
language: zh-CN # 中文博客务必设置，否则界面/日期是英文
timezone: "Asia/Shanghai"
url: https://<你的用户名>.github.io # 重要！影响所有页面链接与资源路径
```

---

## 4. 第二步：安装并启用 Butterfly 主题

Butterfly 是当前最活跃的中文 Hexo 主题之一，界面现代、文档完善、内置搜索/暗色模式/字数统计等。

```bash
npm install hexo-theme-butterfly
npm install hexo-renderer-pug hexo-renderer-stylus --save
```

- 后两个渲染器是 Butterfly 的**必需依赖**（Pug 模板引擎 + Stylus 样式引擎），漏装会导致页面空白。
- 用 npm 安装主题的好处：升级只需 `npm update hexo-theme-butterfly`，且不需要把主题源码拷进仓库。

启用主题：把**根 `_config.yml`** 中 `theme:` 一行改为：

```yaml
theme: butterfly
```

**主题配置的最佳实践**——不要直接修改 `node_modules` 里的主题配置文件，而是把它复制到博客根目录同名的覆盖文件：

```bash
cp node_modules/hexo-theme-butterfly/_config.yml _config.butterfly.yml
```

以后所有主题设置都改 `_config.butterfly.yml`（它优先于主题包内的默认配置，升级主题不会丢失自定义）。

重启预览看效果：

```bash
hexo clean && hexo server
```

---

## 5. 第三步：开启本地全文搜索

共三步，缺一不可。

**1) 安装索引生成插件**

```bash
npm install hexo-generator-searchdb --save
```

**2) 在根目录 `_config.yml` 末尾添加索引配置**

```yaml
search:
  path: search.xml # 索引文件名；写成 search.json 也可以，主题会按扩展名自动识别
  field: post
  content: true # 关键：true 才会把正文纳入索引，否则只能搜标题
  format: striptags
```

Butterfly 会读取这里的 `path` 作为搜索索引的地址，这是索引文件与页面对接的关键。

**3) 在 `_config.butterfly.yml` 中启用**

```yaml
search:
  use: local_search
  # ...其余键保持默认

local_search:
  preload: true # 文章较多时建议 true（提前加载索引，搜索更快）
  top_n_per_article: 1 # 每篇文章最多显示 1 条结果；-1 表示不限
  unescape: false
```

验证：`hexo clean && hexo server`，点击页面右上角搜索图标，输入任意文章关键词。

---

## 6. 第四步：创建 GitHub 仓库并推送

仓库名必须是 `<你的用户名>.github.io`（换成真实用户名，例如 `snjly.github.io`）。

**方式 A（推荐，gh CLI 一条命令完成建仓+推送）**

```bash
cd my-blog
git init -b main
git add .
git commit -m "init: hexo + butterfly blog"

gh repo create <你的用户名>.github.io --public --source=. --remote=origin --push
```

**方式 B（网页手动建仓）**

1. 打开 https://github.com/new：
   - Repository name 填 `<你的用户名>.github.io`
   - 选择 **Public**
   - **不要**勾选任何初始化选项（README / .gitignore / license 都不要）
2. 本地提交后关联并推送：

```bash
git init -b main
git add .
git commit -m "init: hexo + butterfly blog"
git remote add origin https://github.com/<你的用户名>/<你的用户名>.github.io.git
git push -u origin main
```

注意：

- 仓库必须为 **Public**（Private 仓库的 Pages 需要付费计划）。
- `git push` 慢或超时一般是国内网络访问 GitHub 不稳定，重试即可；也可改用 SSH（见 FAQ Q5）。

---

## 7. 第五步：启用 GitHub Actions 自动部署

### 7.1 先在仓库里把 Pages 的“构建源”设为 GitHub Actions

仓库页面 → **Settings → Pages** → Build and deployment → **Source 选择 "GitHub Actions"**。

> 顺序很重要：**先**把 Source 设为 GitHub Actions，**再**添加上一步的工作流文件。这是 Hexo 官方文档特别强调的。

### 7.2 添加部署工作流

在项目里创建 `.github/workflows/pages.yml`，内容如下（基于 Hexo 官方工作流模板，Node 版本改为与本机一致的 24，并加了手动触发）：

```yaml
name: Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          submodules: recursive
      - name: Use Node.js 24
        uses: actions/setup-node@v4
        with:
          node-version: "24"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

提交推送：

```bash
mkdir -p .github/workflows    # 若目录不存在
# 把上面的 YAML 保存为 .github/workflows/pages.yml
git add .github
git commit -m "ci: deploy to github pages via actions"
git push
```

### 7.3 验证

- 打开仓库的 **Actions** 标签页，可看到名为 "Pages" 的工作流开始运行；`build` 与 `deploy` 都打绿勾后，访问 `https://<你的用户名>.github.io`。
- 首次运行约 1–3 分钟。之后每次 `git push` 都会自动重新构建并发布。

---

## 8. 日常写作流程

```bash
hexo new "文章标题"      # 新建文章 → source/_posts/文章标题.md
```

编辑 Markdown，建议写好 front-matter：

```markdown
---
title: 文章标题
date: 2026-10-09 10:00:00
categories: [学习笔记]
tags: [算法, 数据结构]
---

正文……
```

本地预览 → 发布：

```bash
hexo server        # http://localhost:4000
git add . && git commit -m "post: 文章标题" && git push    # 推送即上线
```

学习资料的组织建议：

- `categories` 做**大分类**（保持一个层级，如：学习笔记 / 日志 / 读书），Hexo 会自动生成分类页和归档页。
- `tags` 做**细粒度标签**（如：Python、分布式、论文阅读），可多写。
- 想做“系列/专题”，用固定 tag 或独立分类即可，不需要额外插件。

草稿与独立页面：

```bash
hexo new draft "未完成的草稿"    # 存到 source/_drafts，不会被发布
hexo server --draft              # 预览时包含草稿
hexo publish "未完成的草稿"       # 转正到 _posts

hexo new page about              # 独立页面 → source/about/index.md
```

独立页面建好后，在 `_config.butterfly.yml` 的 `menu:` 中加一行即可出现在导航栏，例如：

```yaml
menu:
  关于: /about/ || fas fa-heart
```

---

## 9. 可选增强（按需再配置）

### 9.1 评论（giscus，基于 GitHub Discussions，免服务器）

1. 给博客仓库开启 Discussions：Settings → General → Features → 勾选 Discussions。
2. 安装 giscus App：https://github.com/apps/giscus （授权到你的博客仓库）。
3. 打开 https://giscus.app ，填入仓库名，页面会生成 `repo_id` 与 `category_id`。
4. 在 `_config.butterfly.yml` 中配置：

```yaml
comments:
  use: giscus
giscus:
  repo: <你的用户名>/<你的用户名>.github.io
  repo_id: <从 giscus.app 获取>
  category_id: <从 giscus.app 获取>
  light_theme: light
  dark_theme: dark
```

### 9.2 自定义域名（以后再做，不影响当前使用）

- 买好域名后，在域名 DNS 处添加 CNAME 记录，指向 `<你的用户名>.github.io`。
- 在仓库 **Settings → Pages → Custom domain** 填入域名，并勾选 Enforce HTTPS。
- 对 Hexo 的正确做法是：在博客项目的 `source/` 目录下放一个名为 `CNAME` 的文件（无扩展名），内容为你的域名（一行）。这样每次自动构建都会带上它，不会被覆盖丢失。
- 根 `_config.yml` 的 `url` 改为 `https://你的域名`。

### 9.3 国内访问优化（可选）

- `*.github.io` 域名在国内访问可能偏慢或间歇不可达（网络环境原因，并非站点故障）。
- 最省事的改善路径：绑定自定义域名；或后续把同一套源码同步部署到 Cloudflare Pages 之类的平台做镜像。建议**先跑通内容**，访问速度后置优化。

### 9.4 主题升级

```bash
npm update hexo-theme-butterfly
```

因为自定义都在根目录 `_config.butterfly.yml`，升级不会覆盖你的设置。若主题新版增加了配置项，可对照 `node_modules/hexo-theme-butterfly/_config.yml` 按需补充到你的覆盖文件中。

---

## 10. 常见问题 FAQ

**Q1：访问 `https://<你的用户名>.github.io` 显示 404？**
依次排查：① 仓库名是否严格等于 `<你的用户名>.github.io`；② Settings → Pages 的 Source 是否已设为 "GitHub Actions"；③ Actions 里最近一次运行是否成功（失败先看日志）；④ 刚部署完等 1–2 分钟再刷新。

**Q2：Actions 构建失败，日志里 npm install 报 404 / 找不到包？**
几乎肯定是 npm 源问题：① 确认 `.npmrc` 已提交（`git ls-files | grep npmrc`）；② 确认 `package-lock.json` 里没有内网源——`grep -c anpm package-lock.json` 必须为 0。若大于 0：删除 `node_modules` 和 `package-lock.json`，重新 `npm install` 后再提交推送。

**Q3：页面样式丢失 / 链接跳到奇怪路径？**
根 `_config.yml` 的 `url` 写错。必须是 `https://<你的用户名>.github.io`（无结尾斜杠、无子路径）。改完提交推送即可。

**Q4：本地 `hexo server` 提示端口被占用？**
换端口：`hexo server -p 4001`。

**Q5：`git push` 很慢或超时？**
国内网络访问 GitHub 不稳定所致。重试；或改用 SSH：先在 GitHub 的 Settings → SSH and GPG keys 添加本机公钥（`ssh-keygen -t ed25519` 生成），再执行：

```bash
git remote set-url origin git@github.com:<你的用户名>/<你的用户名>.github.io.git
```

**Q6：文章没出现在网站或搜索结果里？**
① 检查 front-matter 是否用 `---` 正确包裹且含 `date`；② 搜索需要根配置 `content: true` 且重新构建（CI 会自动做）；③ 强制刷新浏览器（Cmd+Shift+R）排除缓存。

**Q7：想换主题或回退？**
主题只是一个 npm 包 + 一份 `_config.butterfly.yml`。换主题 = 安装新主题包 + 把 `theme:` 改成新主题名，`source/` 下的 Markdown 内容完全不受影响。

---

## 11. 命令速查卡

| 用途                | 命令                                                |
| ------------------- | --------------------------------------------------- |
| 新建文章            | `hexo new "标题"`                                   |
| 新建独立页面        | `hexo new page 页面名`                              |
| 新建草稿 / 发布草稿 | `hexo new draft "x"` / `hexo publish "x"`           |
| 本地预览            | `hexo server`（`-p 4001` 换端口；`--draft` 含草稿） |
| 手动构建            | `npm run build`（即 `hexo generate`）               |
| 清理构建产物        | `hexo clean`                                        |
| 发布上线            | `git add . && git commit -m "..." && git push`      |
| 查看部署状态        | 仓库页面 → Actions 标签页                           |
| 查看 GitHub 用户名  | `gh api user --jq .login`                           |
| 升级主题            | `npm update hexo-theme-butterfly`                   |

---

## 附：本指南执行顺序总览

1. 环境复核 + 全局安装 hexo-cli → 第 2 节
2. `hexo init` 建站 + 切 npm 源 + 本地预览 → 第 3 节
3. 安装 Butterfly + 渲染器 + `_config.butterfly.yml` + 站基础信息 → 第 4 节
4. 装搜索插件 + 两处配置 → 第 5 节
5. 建 `<用户名>.github.io` 仓库并推送 → 第 6 节
6. Pages Source=GitHub Actions + `pages.yml` + 验证上线 → 第 7 节
7. 之后进入日常写作循环 → 第 8 节

建议按顺序执行，每完成一步先本地验证（`hexo clean && hexo server`），再进入下一步。
