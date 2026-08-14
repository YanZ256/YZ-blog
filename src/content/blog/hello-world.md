---
title: '第一篇：你好，世界'
description: '这是我博客的第一篇文章，简单介绍一下这个站点是怎么搭起来的。'
pubDate: 'Aug 14 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

欢迎来到我的博客！这是第一篇文章，用来验证一切正常运行。

## 这个博客是怎么搭的

它基于 [Astro](https://astro.build/) 的官方博客模板：

- 文章用 **Markdown** 写，放在 `src/content/blog/` 目录
- 构建时生成纯静态 HTML，加载快、对 SEO 友好
- 自带 RSS 订阅、站点地图和暗色模式

## 怎么写新文章

在 `src/content/blog/` 下新建一个 `.md` 文件，顶部写好 front matter：

```markdown
---
title: '文章标题'
description: '一句话简介'
pubDate: 'Aug 14 2026'
---

正文从这里开始，用 Markdown 语法书写。
```

保存后刷新页面就能看到。就这么简单，开始写吧！
