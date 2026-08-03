---
title: 博客上线
published: 2026-08-03
description: 0x5t4ckc47 Sec 博客框架已就绪，基于 Astro + Fuwari，并通过 GitHub Actions 部署到 GitHub Pages。
tags: [Blog, Meta]
category: Notes
draft: false
---

# 欢迎来到 0x5t4ckc47 Sec

这是基于 [Fuwari](https://github.com/saicaca/fuwari)（Astro 静态博客模板）搭建的安全向博客。

## 本地开发

```bash
pnpm install
pnpm dev
```

浏览器打开 `http://localhost:4321` 即可预览。

## 新建文章

```bash
pnpm new-post my-post-name
```

文章会生成在 `src/content/posts/`，编辑 frontmatter 与正文后保存即可。

## 部署

推送到 GitHub 仓库 `0x5t4ckc47.github.io` 的 `main` 分支后，GitHub Actions 会自动构建并发布到：

**https://0x5t4ckc47.github.io/**
