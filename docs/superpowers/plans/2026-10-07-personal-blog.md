# Schofi Personal Blog Implementation Plan

> **For agentic workers:** Use subagent-driven-development for the content task and review the integrated result.

**Goal:** Upgrade Schofi's existing welcome page into an English-language, Lil'Log-inspired personal blog published with GitHub Pages.

**Architecture:** Hugo generates static HTML from Markdown. PaperMod is pinned as a Git submodule. Repository-level CSS and a math render hook provide small customizations without editing the upstream theme. GitHub Actions builds the main branch and deploys the generated public directory.

**Tech Stack:** Hugo 0.167.0, PaperMod, Markdown, CSS, GitHub Pages.

## Implementation checklist

- [x] Clone the existing repository, retain its Git history, and work on feat/personal-blog.
- [x] Add the PaperMod submodule and a verified portable Hugo binary in ignored .tools/.
- [x] Configure English metadata, article URLs, search JSON, RSS, navigation, light/dark theme and table of contents in hugo.yaml.
- [x] Add assets/css/extended/custom.css for a narrow reading column, restrained green links, text-first homepage and mobile navigation.
- [x] Add layouts/_markup/render-passthrough.html using Hugo's transform.ToMath with MathML output, plus a local favicon.
- [x] Create content/about.md, content/archives.md, content/search.md, content/posts/_index.md, one welcome article and a draft Markdown sample.
- [x] Create archetypes/posts.md with draft enabled by default and an English README covering writing and deployment.
- [x] Add .github/workflows/hugo.yml with checksum-verified Hugo installation, pull-request build checks and main-branch Pages deployment.
- [x] Run .tools/hugo.exe --gc --minify --panicOnWarning and a separate --buildDrafts build; verify draft exclusion and formula rendering.
- [x] Run a local Hugo server and inspect homepage, article, archive, tags, search, theme toggle and mobile layout in a browser.
- [x] Obtain independent code review, resolve findings, and open the working preview for the user.

## Acceptance criteria

Production output has one original welcome article, working navigation, a search index, RSS and sitemap. Draft material never appears in production output. The English layout remains readable at 390 px and desktop widths. Build output contains no warnings. The site is published at https://schofi.github.io/ through GitHub Actions.
