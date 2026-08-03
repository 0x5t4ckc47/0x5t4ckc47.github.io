# 0x5t4ckc47 Sec

基于 [Astro](https://astro.build) + [Fuwari](https://github.com/saicaca/fuwari) 的静态安全博客。

- 站点：https://0x5t4ckc47.github.io/
- 仓库：`0x5t4ckc47.github.io`
- 部署：GitHub Actions → GitHub Pages

## 环境要求

- Node.js ≥ 20
- pnpm ≥ 9

```bash
# 如未安装 pnpm
npm install -g pnpm
```

## 本地开发

```bash
# 安装依赖
pnpm install

# 启动开发服务器（默认 http://localhost:4321）
pnpm dev

# 生产构建（含 Pagefind 搜索索引）
pnpm build

# 预览构建结果
pnpm preview
```

## 配置说明

| 文件 | 作用 |
|------|------|
| `src/config.ts` | 站点标题、语言、导航、侧栏资料、主题色等 |
| `astro.config.mjs` | `site` / `base`、集成与 Markdown 插件 |
| `.github/workflows/deploy.yml` | 推送到 `main` 后自动部署 GitHub Pages |

当前关键配置：

```js
// astro.config.mjs
site: "https://0x5t4ckc47.github.io"
base: "/"   // User Pages 根站，仓库名必须是 0x5t4ckc47.github.io
```

```ts
// src/config.ts
title: "0x5t4ckc47 Sec"
lang: "zh_CN"
```

## 写文章

```bash
pnpm new-post <filename>
```

在 `src/content/posts/<filename>.md` 中编辑：

```yaml
---
title: 文章标题
published: 2026-08-03
description: 摘要
image: ./cover.jpg   # 可选封面
tags: [Tag1, Tag2]
category: Notes
draft: false
---
```

About 页面内容：`src/content/spec/about.md`

## GitHub Pages 部署

### 1. 创建远程仓库

仓库名必须为：

```text
0x5t4ckc47.github.io
```

### 2. 推送代码

```bash
git init
git add .
git commit -m "init: Astro + Fuwari blog"
git branch -M main
git remote add origin git@github.com:0x5t4ckc47/0x5t4ckc47.github.io.git
git push -u origin main
```

### 3. 打开 GitHub Pages

1. 打开仓库 **Settings → Pages**
2. **Source** 选择 **GitHub Actions**
3. 等待 Actions 中的 **Deploy to GitHub Pages** 成功
4. 访问 https://0x5t4ckc47.github.io/

也可在 Actions 页手动运行 `Deploy to GitHub Pages` workflow。

### 工作流说明

| Workflow | 作用 |
|----------|------|
| `deploy.yml` | 构建并部署到 GitHub Pages（正式上线） |
| `build.yml` | PR / push 时做构建与 `astro check`（质量检查） |
| `biome.yml` | 代码风格检查 |

## 常用命令

| 命令 | 说明 |
|------|------|
| `pnpm install` | 安装依赖 |
| `pnpm dev` | 本地开发 |
| `pnpm build` | 生产构建 |
| `pnpm preview` | 预览构建 |
| `pnpm check` | Astro 类型/内容检查 |
| `pnpm new-post <name>` | 新建文章 |
| `pnpm format` | Biome 格式化 |

## 自定义域名（可选）

若以后使用自定义域名：

1. 在 `public/CNAME` 写入域名（一行）
2. 将 `astro.config.mjs` 的 `site` 改为该域名
3. 保持 `base: "/"`（不要设置仓库路径）
4. 在域名 DNS 按 GitHub Pages 文档配置

## 主题来源

本项目基于 [saicaca/fuwari](https://github.com/saicaca/fuwari)（MIT License）。
