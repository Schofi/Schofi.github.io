# Schofi's Blog

基于 **Hugo 0.167.0 + PaperMod** 的英文个人博客，发布到 <https://schofi.github.io/>。作者为 xiangxin，内容以 AI、工程实践和学习记录为主。以下维护说明保留中文。

## 本地预览

主题通过 Git 子模块管理。首次克隆时执行：

```powershell
git clone --recurse-submodules https://github.com/Schofi/Schofi.github.io.git
cd Schofi.github.io
```

如果已经克隆过仓库，在仓库目录执行：

```powershell
git submodule update --init --recursive
```

当前工作区如果已有便携版 Hugo，可以直接启动：

```powershell
.\.tools\hugo.exe server
```

`.tools` 中的本地工具不随仓库分发。在其他电脑上安装 [Hugo 0.167.0](https://github.com/gohugoio/hugo/releases/tag/v0.167.0)，将其加入 `PATH` 后，也可以使用：

```powershell
hugo server
```

打开终端显示的预览地址，通常是 <http://localhost:1313/>。保持终端运行，修改文章后页面会自动刷新，按 `Ctrl+C` 停止预览。

查看草稿时添加 `--buildDrafts`：

```powershell
hugo server --buildDrafts
```

下文命令中的 `hugo` 均可替换成 `.\.tools\hugo.exe`。

## 写文章

创建一篇独立文章：

```powershell
hugo new content posts/my-first-post.md
```

如果文章需要图片，推荐创建一个文章目录：

```powershell
hugo new content posts/my-note/index.md
```

将图片放在 `content/posts/my-note/` 中，在正文通过 `![图片说明](image.png)` 引用。

新文章使用 `archetypes/posts.md` 模板，默认 `draft: true`。编辑文件顶部的标题、日期、摘要和标签，再填写正文：

```yaml
title: "文章标题"
date: 2026-10-07T09:00:00+08:00
draft: true
author: "xiangxin"
description: "搜索引擎和分享卡片使用的简短说明。"
summary: "文章列表显示的摘要。"
tags: ["大模型", "工程实践"]
ShowToc: true
```

数学公式会自动渲染，无需在文章顶部添加开关。行内公式使用 `$...$`，独立公式使用 `$$...$$`；Hugo 在构建时将公式转换为 MathML，无需加载外部公式脚本。正文中的普通美元符号需写作 `\\$`（两个反斜杠加美元符号），避免被识别为公式边界；代码块中的美元符号保持原样。写法与排版可参考 `content/posts/writing-example.md`；该示例默认是草稿。

准备发布时，将 `draft: true` 改为 `draft: false`。未来日期的文章在日期到达前默认不会生成；需要检查时可以在本地增加 `--buildFuture`。

## 检查与发布

先验证正式构建：

```powershell
hugo --gc --minify
```

生成的网站位于 `public/`。此目录是构建产物，无需提交。

在 GitHub 仓库中进入 **Settings → Pages → Build and deployment → Source**，选择 **GitHub Actions**。将改动提交并推送到 `main` 分支后，`.github/workflows/hugo.yml` 会使用 Hugo 0.167.0 构建并部署网站，也可以在 **Actions** 页面手动运行该工作流。

第一次启用或部署失败时，到 GitHub 仓库的 **Actions** 页面检查构建日志。部署成功后访问 <https://schofi.github.io/>。

## 常用内容位置

| 位置 | 用途 |
| --- | --- |
| `content/posts/` | 文章与文章配图 |
| `content/about.md` | 关于页面 |
| `content/archives.md` | 归档页面 |
| `content/search.md` | 搜索页面 |
| `archetypes/posts.md` | 新文章模板 |
| `hugo.yaml` | 站点配置 |
| `themes/PaperMod/` | PaperMod 主题子模块 |
| `.github/workflows/hugo.yml` | GitHub Pages 构建与部署 |

站名、首页介绍、菜单和社交链接在 `hugo.yaml` 中修改。定制样式和模板位于站点自身的 `assets/`、`layouts/` 中，避免直接修改主题子模块。

`layouts/baseof.html`、`layouts/rss.xml` 和 `layouts/_partials/templates/opengraph.html` 保留了 PaperMod 原始模板，仅将旧语言属性换成 Hugo 0.167.0 的 `Direction` / `Locale`。更新主题时应一并检查这三个兼容模板。

## 更新主题

主题固定在一个 Git 提交上，更新时执行：

```powershell
git submodule update --remote --merge themes/PaperMod
hugo --gc --minify
```

本地检查首页、文章、归档、搜索及手机布局后，提交 `themes/PaperMod` 的子模块指针变更。若新版本有不兼容改动，先处理再发布。

## 致谢

博客的阅读体验参考 [Lil’Log](https://lilianweng.github.io/)，由 [Hugo](https://gohugo.io/) 和 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 提供支持。本站文章独立撰写。
