# Schofi's Blog

An English-language personal blog built with **Hugo 0.167.0 + PaperMod**, published at <https://schofi.github.io/>. Written by xiangxin, it covers AI, engineering, and notes from ongoing learning.

## Local preview

The theme is managed as a Git submodule. To clone the site:

```powershell
git clone --recurse-submodules https://github.com/Schofi/Schofi.github.io.git
cd Schofi.github.io
```

If you have already cloned the repository, initialize the theme from the repository directory:

```powershell
git submodule update --init --recursive
```

If the portable Hugo binary is available in your workspace, start the preview with:

```powershell
.\.tools\hugo.exe server
```

Local tools in `.tools/` are not included in the repository. On another computer, install [Hugo 0.167.0](https://github.com/gohugoio/hugo/releases/tag/v0.167.0), add it to your `PATH`, and run:

```powershell
hugo server
```

Open the preview URL shown in the terminal, usually <http://localhost:1313/>. Keep the terminal running to see changes as you edit. Press `Ctrl+C` to stop the server.

To preview drafts:

```powershell
hugo server --buildDrafts
```

In the commands below, you can replace `hugo` with `.\.tools\hugo.exe`.

## Writing posts

Create a standalone post:

```powershell
hugo new content posts/my-first-post.md
```

For posts with images, use a page bundle:

```powershell
hugo new content posts/my-note/index.md
```

Place images in `content/posts/my-note/` and reference them with `![Image description](image.png)`.

New posts use `archetypes/posts.md` and have `draft: true` by default. Edit the title, date, summary, and tags in the front matter, then write the body:

```yaml
title: "Post title"
date: 2026-10-07T09:00:00+08:00
draft: true
author: "xiangxin"
description: "A short description for search engines and link previews."
summary: "A summary displayed in post listings."
tags: ["LLM", "Engineering"]
ShowToc: true
```

Math is rendered automatically; no front matter switch is needed. Use `$...$` for inline equations and `$$...$$` for display equations. Hugo converts them to MathML during the build, so readers do not need an external math script. For a literal dollar sign in prose, write `\\$` in the Markdown source (two backslashes followed by a dollar sign). Dollar signs in code blocks can be written normally. See `content/posts/writing-example.md` for formatting examples; it remains a draft by default.

When a post is ready, change `draft: true` to `draft: false`. Future-dated posts are excluded until their publication date. Add `--buildFuture` to a local preview command to inspect them early.

## Building and publishing

Check the production build first:

```powershell
hugo --gc --minify --panicOnWarning
```

Generated files are written to `public/`. This directory contains build output and should not be committed.

In the GitHub repository, open **Settings → Pages → Build and deployment → Source** and select **GitHub Actions**. Commits pushed to `main` trigger `.github/workflows/hugo.yml`, which builds the site with Hugo 0.167.0 and deploys it. You can also start the workflow manually from the **Actions** tab.

If deployment fails, inspect the workflow logs in **Actions**. Once deployment succeeds, visit <https://schofi.github.io/>.

## Content and configuration

| Location | Purpose |
| --- | --- |
| `content/posts/` | Posts and their images |
| `content/about.md` | About page |
| `content/archives.md` | Archive page |
| `content/search.md` | Search page |
| `content/tags/notes/_index.md` | Notes tag and a redirect for its previous URL |
| `archetypes/posts.md` | Template for new posts |
| `hugo.yaml` | Site configuration |
| `themes/PaperMod/` | PaperMod theme submodule |
| `.github/workflows/hugo.yml` | GitHub Pages build and deployment |

Edit `hugo.yaml` to change the site title, home introduction, menu, or social links. The site uses English for its interface, content, dates, search index, and RSS feed. Custom styles and templates live in the site's `assets/` and `layouts/` directories, rather than inside the theme submodule.

`layouts/baseof.html`, `layouts/rss.xml`, and `layouts/_partials/templates/opengraph.html` retain the original PaperMod templates with deprecated language properties replaced by Hugo 0.167.0's `Direction` and `Locale`. Review these three compatibility overrides when updating the theme.

## Updating the theme

The theme is pinned to a Git commit. To update it:

```powershell
git submodule update --remote --merge themes/PaperMod
hugo --gc --minify --panicOnWarning
```

Check the homepage, posts, archive, search, and mobile layout locally, then commit the updated `themes/PaperMod` submodule reference. Resolve any compatibility issues before publishing.

## Credits

The reading experience is inspired by [Lil’Log](https://lilianweng.github.io/). The site is powered by [Hugo](https://gohugo.io/) and [PaperMod](https://github.com/adityatelange/hugo-PaperMod). Blog content is written independently.
