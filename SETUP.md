# 从零搭建并部署个人博客（Astro + Cloudflare Pages）

本文记录本博客从零搭建到上线的完整流程，方便日后维护、迁移或重建。

- 本地项目目录：`/home/zy63/Documents/BLOG`
- GitHub 仓库：<https://github.com/YanZ256/YZ-blog>
- 线上地址：<https://yz-blog.pages.dev>
- 技术栈：Astro 5（静态站点生成）+ Cloudflare Pages（托管 + 全球 CDN + 自动 HTTPS）

---

## 一、整体架构

```
本地写文章 (Markdown)
        │  git push
        ▼
GitHub 仓库 (YZ-blog，存源码)
        │  Cloudflare 检测到 push
        ▼
Cloudflare Pages 自动拉代码 → npm run build → 发布 dist/
        ▼
https://yz-blog.pages.dev  （全球可访问）
```

核心理念：**内容是纯文本 Markdown，构建生成静态 HTML，托管在免费 CDN 上。** 写完 `git push`，上线全自动（这套流程叫 CI/CD）。

---

## 二、环境要求

- Node.js（本项目用 v20.13.1；注意 Astro 7 需要 Node ≥ 22，故本项目固定用 Astro 5 以兼容 Node 20）
- npm
- git
- 一个 GitHub 账号（已配置 SSH key，可免密推送）
- 一个 Cloudflare 账号（用 GitHub 登录即可）

检查环境：

```bash
node -v
npm -v
git --version
```

---

## 三、本地搭建步骤

### 1. 获取 Astro 官方博客模板

用 `giget` 直接拉取官方 blog 示例到目标目录（比走交互式 `npm create astro` 更稳定）：

```bash
npx --yes giget@latest github:withastro/astro/examples/blog /home/zy63/Documents/BLOG --force
```

> 注意：访问 GitHub 较慢时，这一步可能耗时较久，耐心等待即可。

### 2. 固定依赖版本到 Astro 5（兼容 Node 20）

模板默认可能拉取需要 Node 22 的 Astro 7。编辑 `package.json`，将依赖固定为 Astro 5 系列：

```json
{
  "name": "my-blog",
  "type": "module",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "astro": "astro"
  },
  "dependencies": {
    "@astrojs/mdx": "^4.2.0",
    "@astrojs/rss": "^4.0.11",
    "@astrojs/sitemap": "^3.2.1",
    "astro": "^5.13.0",
    "sharp": "^0.33.5"
  }
}
```

### 3. 安装依赖

```bash
npm install --prefix /home/zy63/Documents/BLOG
```

验证 Astro 版本：

```bash
/home/zy63/Documents/BLOG/node_modules/.bin/astro --version
# 期望输出 astro v5.x
```

### 4. 本地化内容

- `src/consts.ts`：站点标题与描述

  ```ts
  export const SITE_TITLE = '我的博客';
  export const SITE_DESCRIPTION = '记录学习、思考与生活。';
  ```

- `src/pages/index.astro`：首页 `<html lang="en">` 改为 `lang="zh"`，正文换成中文。
- `src/content/blog/`：删掉英文示例（`first-post.md`、`second-post.md`、`third-post.md`、`using-mdx.mdx`、`markdown-style-guide.md`），新建自己的中文文章。

### 5. 配置 astro.config.mjs

关键点：

1. Astro 5 的本地字体是**实验特性**，`fonts` 必须放在 `experimental` 内，且字体变体在 `options.variants` 里。
2. `site` 部署后填写正式网址（影响 RSS 与 sitemap 的绝对链接）。

```js
// @ts-check
import mdx from '@astrojs/mdx';
import sitemap from '@astrojs/sitemap';
import { defineConfig, fontProviders } from 'astro/config';

export default defineConfig({
  site: 'https://yz-blog.pages.dev',
  integrations: [mdx(), sitemap()],
  experimental: {
    fonts: [
      {
        provider: fontProviders.local(),
        name: 'Atkinson',
        cssVariable: '--font-atkinson',
        fallbacks: ['sans-serif'],
        options: {
          variants: [
            {
              src: ['./src/assets/fonts/atkinson-regular.woff'],
              weight: 400,
              style: 'normal',
              display: 'swap',
            },
            {
              src: ['./src/assets/fonts/atkinson-bold.woff'],
              weight: 700,
              style: 'normal',
              display: 'swap',
            },
          ],
        },
      },
    ],
  },
});
```

### 6. 修正模板的 BaseHead URL bug

模板 `src/components/BaseHead.astro` 在静态构建时用 `Astro.url` 拼绝对 URL 会报 `Invalid URL`，改用 `Astro.site` / `canonicalURL`：

```astro
<meta property="og:url" content={canonicalURL} />
<meta property="og:image" content={new URL(image.src, Astro.site)} />
```

### 7. 本地构建验证

```bash
npm run build --prefix /home/zy63/Documents/BLOG
```

看到 `[build] Complete!` 即成功，产物在 `dist/`。

### 8. 本地预览

```bash
cd /home/zy63/Documents/BLOG
npm run preview   # 静态预览已构建的 dist，不监听文件
# 打开 http://localhost:4321/
```

> `npm run dev`（热更新开发模式）会监听大量文件。如果遇到
> `ENOSPC: System limit for number of file watchers reached`，
> 是系统 inotify 监听数配额被占满（常见于同时打开大型代码库时）。
> 解决（需 sudo，按需执行）：
>
> ```bash
> sudo sysctl fs.inotify.max_user_watches=524288
> echo 'fs.inotify.max_user_watches=524288' | sudo tee -a /etc/sysctl.conf
> sudo sysctl -p
> ```
>
> 只是写博客的话，用 `npm run preview` 已足够。

---

## 四、推送到 GitHub

### 1. 在 GitHub 网页建一个空仓库

- 打开 <https://github.com/new>
- 名称如 `YZ-blog`，Public（公开仓库 ≠ 别人可写，写权限只属于你）
- **不要**勾选任何初始化文件（README / .gitignore / license）

### 2. 本地关联并推送

```bash
cd /home/zy63/Documents/BLOG
git init -b main            # 若尚未初始化
git add -A
git commit -m "init: Astro blog scaffold with Chinese localization"
git remote add origin git@github.com:YanZ256/YZ-blog.git
git push -u origin main
```

> 使用 SSH 地址（`git@github.com:...`）+ 已配置的 SSH key，可免密推送。
> 验证 SSH：`ssh -T git@github.com`，出现 `Hi <用户名>!` 即通。

---

## 五、部署到 Cloudflare Pages

### 1. 登录

- 打开 <https://dash.cloudflare.com>，用 **GitHub 账号登录**（后续连仓库更顺）。
- 说明：Cloudflare 为境外服务，国内访问控制面板可能不稳定，网络不佳时需自备稳定网络环境。

### 2. 创建 Pages 项目

1. 左侧菜单 **计算 / Workers 和 Pages**
2. 页面底部 **「想要部署 Pages? 开始使用」**（走纯 Pages 流程，配置更清晰）
3. 选 **「导入现有 Git 存储库」→ 开始使用**
4. 首次需授权 Cloudflare 访问 GitHub：选 **Only select repositories**，只勾 `YZ-blog`（最小授权），点 **Install & Authorize**

### 3. 构建配置（关键）

| 字段 | 值 |
|------|----|
| 项目名称 | `yz-blog`（决定网址前缀 `yz-blog.pages.dev`）|
| 生产分支 | `main` |
| 框架预设 | Astro |
| 构建命令 | `npm run build` |
| 构建输出目录 | `dist` |

点 **保存并部署**，等待 1–3 分钟（首次 `npm install` 较慢），出现「成功！您的项目已部署到全球」即完成。

### 4. 收尾：设置正式 site 地址

拿到正式网址后，把 `astro.config.mjs` 的 `site` 改成 `https://yz-blog.pages.dev`，然后：

```bash
git add astro.config.mjs
git commit -m "chore: set site url to yz-blog.pages.dev"
git push
```

Cloudflare 检测到 push 会**自动重新部署**。

---

## 六、日常维护

### 发布新文章

1. 在 `src/content/blog/` 新建 `.md` 文件，写好 front matter：

   ```markdown
   ---
   title: '文章标题'
   description: '一句话简介'
   pubDate: 'Aug 15 2026'
   ---

   正文用 Markdown 书写……
   ```

2. 推送即上线：

   ```bash
   cd /home/zy63/Documents/BLOG
   git add -A
   git commit -m "post: 新文章标题"
   git push
   ```

3. 等 1–2 分钟，Cloudflare 自动构建部署，刷新网站即可看到。

### 常改的文件

| 想改什么 | 文件 |
|---------|------|
| 站点标题 / 描述 | `src/consts.ts` |
| 首页内容 | `src/pages/index.astro` |
| 导航栏 | `src/components/Header.astro` |
| 页脚署名 | `src/components/Footer.astro` |
| 关于页 | `src/pages/about.astro` |
| 配色风格 | `src/styles/global.css` |
| 文章 front matter schema | `src/content.config.ts` |

---

## 七、可选优化

- **绑定自定义域名**：Cloudflare Pages 项目 → 自定义域，绑定自己的域名（需另买域名）。
- **访问统计**：Cloudflare 项目里开启 Web Analytics，会得到一段 beacon 脚本。
- **评论系统**：可接入 Giscus（基于 GitHub Discussions，免费、零后端）。

---

## 八、踩过的坑（排错备忘）

| 现象 | 原因 | 解决 |
|------|------|------|
| `Node.js vXX is not supported by Astro` | 模板默认拉了 Astro 7（需 Node 22）| 固定 `astro` 到 `^5.13.0` 后重装 |
| `ExperimentalFontsNotEnabled` | 字体是实验特性 | `fonts` 移入 `experimental`，变体放 `options.variants` |
| 构建时 `Invalid URL` | BaseHead 用 `Astro.url` 拼绝对地址 | 改用 `canonicalURL` / `Astro.site`，并正确设置 `site` |
| `ENOSPC: file watchers reached` | 系统 inotify 配额占满 | 用 `npm run preview`，或调高 `fs.inotify.max_user_watches` |
| `dash.cloudflare.com` 打不开 | 境外服务，网络不稳 | 换浏览器 / 无痕 / 自备稳定网络 |

---

*文档记录时间：2026-08-14*
