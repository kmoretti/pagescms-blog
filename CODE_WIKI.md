# Code Wiki — AcoFork Feed 博客项目

> 本文档为基于 Eleventy (11ty) v2.0 的个人静态博客项目 `AcoFork Feed` 的完整代码维基，涵盖项目整体架构、主要模块职责、关键类与函数说明、依赖关系以及项目运行方式等关键信息。

---

## 目录

1. [项目概述](#1-项目概述)
2. [项目整体架构](#2-项目整体架构)
3. [目录结构](#3-目录结构)
4. [核心模块职责](#4-核心模块职责)
5. [关键配置文件说明](#5-关键配置文件说明)
6. [关键类与函数说明](#6-关键类与函数说明)
7. [模板系统](#7-模板系统)
8. [数据流与构建流程](#8-数据流与构建流程)
9. [依赖关系](#9-依赖关系)
10. [项目运行方式](#10-项目运行方式)
11. [部署方式](#11-部署方式)
12. [内容创作指南](#12-内容创作指南)
13. [定制与扩展点](#13-定制与扩展点)

---

## 1. 项目概述

### 1.1 项目简介

| 属性 | 值 |
|---|---|
| 项目名称 | AcoFork Feed（基于 eleventy-base-blog v8） |
| 站点 URL | https://feed.2x.nz/ |
| 作者 | 二叉树树（acofork@qq.com） |
| 语言 | 简体中文（zh-CN） |
| 站点描述 | "一个随时都会冒出新奇想法的神奇小站" |
| 静态站点生成器 | Eleventy (11ty) v2.0 |
| 模板引擎 | Nunjucks（.njk） + Markdown（.md） |
| 文章数量 | 约 100 篇（位于 `content/blog/`） |
| 许可证 | MIT |

### 1.2 项目定位

这是一个**纯预渲染静态博客**（零客户端 JS 框架输出），基于 Eleventy 官方的 `eleventy-base-blog` 起始模板进行深度中文化与功能定制。所有 HTML 在构建时生成，性能优异（Lighthouse 四项满分），并通过 PagesCMS 实现免本地环境的在线内容编辑。

### 1.3 核心特性

- **零 JavaScript 框架输出**：内容完全预渲染，仅含少量原生 JS（灯箱、搜索）
- **性能优先**：开箱即用的 Lighthouse 四项满分
- **图片优化**：通过 `{% image %}` 短代码自动生成 AVIF/WebP 等现代格式
- **草稿支持**：`draft: true` 标记的文章仅在 `--serve`/`--watch` 模式下可见
- **置顶文章**：`pin: true` 的文章排在列表最前，并带渐变动画样式
- **封面图支持**：列表页与搜索结果展示封面图
- **图片灯箱**：点击正文图片可放大查看，支持滚轮缩放、拖拽平移
- **Giscus 评论**：基于 GitHub Discussions 的评论系统
- **客户端搜索**：基于 Atom Feed 的全文搜索
- **GFM 警报块**：支持 `> [!tip|info|warning|caution]` 语法
- **外链自动 `target="_blank"`**：构建时自动处理外链
- **RSS / JSON Feed**：自动生成 Atom 与 JSON 订阅源
- **sitemap.xml**：自动生成站点地图
- **零维护标签页**：自动为每个标签生成独立页面
- **PagesCMS 集成**：通过 `.pages.yml` 配置在线 CMS 编辑内容

---

## 2. 项目整体架构

### 2.1 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    Eleventy 构建核心                         │
│                  (eleventy.config.js)                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐  ┌─────────────────┐  ┌──────────────┐
│  内容源        │  │  模板/布局       │  │  全局数据     │
│  content/      │  │  _includes/      │  │  _data/       │
│  - blog/*.md   │  │  - layouts/      │  │  - metadata.js│
│  - about/      │  │    base.njk      │  └──────────────┘
│  - feed/       │  │    home.njk      │
│  - sitemap/    │  │    post.njk      │
│  - index.njk   │  │  - postslist.njk │
│  - 404.md      │  └─────────────────┘
│  - search.njk  │
│  - tags*.njk   │
└───────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│              插件层 (Plugins)                                │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐  │
│  │ pluginDrafts │ │ pluginImages │ │  pluginRss         │  │
│  │ (草稿管理)    │ │ (图片优化)    │ │  (RSS 过滤器)      │  │
│  └──────────────┘ └──────────────┘ └────────────────────┘  │
│  ┌─────────────────────┐ ┌──────────────────────────────┐  │
│  │ pluginSyntaxHighlight│ │ pluginNavigation             │  │
│  │ (代码高亮)            │ │ (导航菜单)                   │  │
│  └─────────────────────┘ └──────────────────────────────┘  │
│  ┌──────────────────────┐ ┌────────────────────────────┐   │
│  │ EleventyHtmlBasePlugin│ │ pluginBundle              │   │
│  │ (子目录路径支持)       │ │ (CSS/JS 打包)              │   │
│  └──────────────────────┘ └────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              自定义 Filters & Transforms                     │
│  Filters: readableDate / htmlDateString / sortByPin /        │
│           head / min / getAllTags / filterTagList            │
│  Transforms: externalBlank / alertBlocks                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              静态输出 _site/                                  │
│  HTML / CSS / img / feed.xml / feed.json / sitemap.xml       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 架构特点

1. **配置即代码**：所有构建逻辑集中在 `eleventy.config.js`，通过插件、过滤器、Transform 组合实现功能
2. **数据驱动**：通过 [Eleventy Data Cascade](https://www.11ty.dev/docs/data-cascade/) 机制，模板可访问全局数据、目录数据、Frontmatter 数据
3. **关注点分离**：内容（`content/`）、模板（`_includes/`）、数据（`_data/`）、静态资源（`public/`）严格分层
4. **零运行时依赖**：构建产物为纯静态文件，可托管于任意静态托管服务

---

## 3. 目录结构

```
workspace/
├── _data/                          # 全局数据目录
│   └── metadata.js                 # 站点元数据（标题、URL、作者等）
├── _includes/                       # 模板包含目录
│   ├── layouts/                     # 布局模板
│   │   ├── base.njk                 # 基础 HTML 结构（所有页面父布局）
│   │   ├── home.njk                 # 首页/列表页布局
│   │   └── post.njk                 # 文章详情页布局
│   └── postslist.njk                # 文章列表复用组件
├── content/                         # 内容输入目录（Eleventy input）
│   ├── about/
│   │   └── index.md                 # 关于页面
│   ├── blog/                        # 博客文章目录（约 100 篇 .md）
│   │   ├── blog.11tydata.js         # blog 目录级数据（默认 tags/布局/permalink）
│   │   ├── ai-agent.md
│   │   ├── obsidian.md
│   │   ├── pin.md                   # 置顶示例文章（加群向导）
│   │   ├── first.md                 # 第一篇 PagesCMS 文章
│   │   └── ... (100 篇)
│   ├── feed/
│   │   ├── feed.11tydata.js         # feed 目录级数据（排除出集合）
│   │   ├── feed.njk                 # Atom Feed (feed.xml)
│   │   └── json.njk                 # JSON Feed (feed.json)
│   ├── sitemap/
│   │   └── sitemap.xml.njk          # 站点地图 sitemap.xml
│   ├── 404.md                       # 404 页面
│   ├── index.njk                    # 首页
│   ├── posts.njk                    # /posts/ 跳转页（重定向到首页）
│   ├── search.njk                   # 搜索页
│   ├── tags-list.njk                # 标签列表页 /tags/
│   └── tags.njk                     # 单个标签页 /tags/{tag}/
├── public/                          # 静态资源目录（passthrough copy）
│   ├── css/
│   │   ├── index.css                # 主样式表
│   │   └── prism-diff.css           # Prism diff 语法高亮样式
│   └── img/                         # 文章图片资源（数百张 webp/png）
├── eleventy.config.js               # Eleventy 主配置文件
├── eleventy.config.drafts.js        # 草稿插件配置
├── eleventy.config.images.js        # 图片短代码插件配置
├── package.json                     # 项目依赖与脚本
├── netlify.toml                     # Netlify 部署配置（含 Lighthouse 插件）
├── .pages.yml                       # PagesCMS 在线 CMS 配置
├── .nvmrc                           # Node 版本约束（16）
├── .editorconfig                    # 编辑器配置（tab 缩进，LF 换行）
├── .gitignore                       # Git 忽略规则
├── .nojekyll                        # 禁用 GitHub Pages 的 Jekyll 处理
├── LICENSE                          # MIT 许可证
└── README.md                        # 项目说明（基于官方模板）
```

### 3.1 输出目录（构建产物）

构建后生成 `_site/` 目录，结构示例：

```
_site/
├── index.html                       # 首页
├── about/index.html                 # 关于页
├── posts/{fileSlug}/index.html      # 各文章详情页
├── tags/index.html                  # 标签列表页
├── tags/{tag}/index.html            # 各标签页
├── search/index.html                # 搜索页
├── 404.html                         # 404 页
├── feed/
│   ├── feed.xml                     # Atom Feed
│   └── feed.json                    # JSON Feed
├── sitemap.xml                      # 站点地图
├── public/                          # 静态资源原样复制
│   ├── css/
│   └── img/
└── img/                             # 图片优化产物（AVIF/WebP）
```

---

## 4. 核心模块职责

### 4.1 配置模块

#### `eleventy.config.js` — 主配置入口

**职责**：Eleventy 项目的中枢配置文件，负责：
- 注册所有官方与自定义插件
- 配置静态资源 Passthrough Copy
- 注册自定义 Filters（日期格式化、置顶排序、标签过滤等）
- 注册自定义 Transforms（外链处理、GFM 警报块）
- 配置 Markdown 渲染选项（linkify、heading anchor）
- 定义模板格式、目录结构、输出路径

#### `eleventy.config.drafts.js` — 草稿管理插件

**职责**：实现草稿文章的环境感知渲染：
- 通过 `addGlobalData` 注入计算属性 `eleventyComputed.permalink` 和 `eleventyComputed.eleventyExcludeFromCollections`
- 在 `--serve`/`--watch` 模式下，自动设置 `BUILD_DRAFTS=true`，草稿文章可见
- 在生产构建时，`draft: true` 的文章 `permalink` 返回 `false`（不输出文件），并从集合中排除

#### `eleventy.config.images.js` — 图片优化插件

**职责**：注册 `{% image %}` 异步短代码：
- 支持本地相对路径与完整 URL 图片
- 自动生成 AVIF、WebP、auto（原格式）三种格式
- 输出到 `_site/img/` 目录
- 自动添加 `loading="lazy"`、`decoding="async"` 属性
- 通过 `eleventyImage.generateHTML` 生成 `<picture>` 或 `<img>` 标记

### 4.2 数据模块

#### `_data/metadata.js` — 站点元数据

**职责**：提供全局可访问的站点元信息，所有模板通过 `metadata.*` 访问：

```javascript
module.exports = {
  title: "AcoFork Feed",
  url: "https://feed.2x.nz/",
  language: "zh-CN",
  description: "AcoFork Feed。一个随时都会冒出新奇想法的神奇小站",
  icon: "https://q2.qlogo.cn/headimg_dl?dst_uin=2726730791&spec=0",
  author: {
    name: "二叉树树",
    email: "acofork@qq.com",
    url: "https://github.com/afoim"
  }
}
```

#### `content/blog/blog.11tydata.js` — blog 目录级数据

**职责**：为 `content/blog/` 下所有文章注入默认数据，实现"约定优于配置"：

```javascript
module.exports = {
  tags: ["posts"],                    // 自动加入 posts 集合
  layout: "layouts/post.njk",         // 默认使用 post 布局
  eleventyComputed: {
    permalink: data => `/posts/${data.page.fileSlug}/`  // 用文件名作为 URL
  }
};
```

**关键作用**：作者只需在 `content/blog/` 下创建 `.md` 文件并填写 Frontmatter，无需重复声明 layout、tags、permalink。

#### `content/feed/feed.11tydata.js` — feed 目录级数据

**职责**：将 feed 模板从 collections 中排除，避免污染文章列表。

### 4.3 模板模块

详见 [第 7 节：模板系统](#7-模板系统)。

### 4.4 静态资源模块

#### `public/css/index.css` — 主样式表

**职责**：定义全站样式，特点：
- 暗色主题（`--background-color: #15202b`）
- 通过 CSS 变量统一管理颜色与字体
- 最大宽度 `40em` 的阅读优化布局
- 包含导航、文章列表、文章详情、灯箱等样式

#### `public/css/prism-diff.css` — Diff 高亮样式

**职责**：为 Prism 代码高亮插件提供 diff 语法（`+`/`-` 行）的样式。

### 4.5 内容管理模块

#### `.pages.yml` — PagesCMS 配置

**职责**：配置 [PagesCMS](https://pagescms.org) 在线 CMS 集成，允许通过 Web 界面：
- 编辑博客文章（`content/blog/`）：标题、描述、封面图、日期、草稿、置顶、标签、正文
- 编辑关于页（`content/about/index.md`）
- 编辑元数据（`_data/metadata.js`）
- 触发部署工作流（Deploy to Cloudflare）
- 触发文章广播工作流（推送通知到 QQ 群和邮件订阅）

---

## 5. 关键配置文件说明

### 5.1 `package.json`

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | eleventy-base-blog | 沿用起始模板名 |
| `version` | 8.0.0 | 模板版本 |
| `engines.node` | >=14 | Node 最低版本（实际 .nvmrc 指定 16） |
| `scripts.build` | `npx @11ty/eleventy` | 生产构建 |
| `scripts.build-ghpages` | `npx @11ty/eleventy --pathprefix=/eleventy-base-blog/` | GitHub Pages 子路径构建 |
| `scripts.start` | `npx @11ty/eleventy --serve --quiet` | 本地开发服务器（含热重载） |
| `scripts.debug` | `DEBUG=Eleventy* npx @11ty/eleventy` | 调试模式 |
| `scripts.benchmark` | `DEBUG=Eleventy:Benchmark* npx @11ty/eleventy` | 性能基准测试 |

### 5.2 `.nvmrc`

```
16
```

指定 Node.js 16 作为推荐运行时版本（nvm 自动切换）。

### 5.3 `.editorconfig`

统一编辑器配置：
- 缩进风格：tab，宽度 2
- 换行符：LF
- 文件末尾插入空行
- 去除行尾空白
- 字符集：UTF-8

### 5.4 `netlify.toml`

Netlify 部署配置：
- 发布目录：`_site`
- 构建命令：`npm run build`
- 启用 `@netlify/plugin-lighthouse` 插件
- Lighthouse 阈值：性能/可访问性/最佳实践/SEO 均为 1.0（满分）
- 报告输出路径：`reports/lighthouse/index.html`

### 5.5 `.gitignore`

```
_site/              # 构建产物
node_modules/       # 依赖
package-lock.json   # 锁文件（注意：不锁定依赖版本）
.cache              # Eleventy 缓存
```

### 5.6 `.nojekyll`

空文件，告知 GitHub Pages 跳过 Jekyll 处理，直接发布 `_site` 内容（保留下划线开头的目录如 `_data` 不会被 Jekyll 忽略）。

---

## 6. 关键类与函数说明

### 6.1 `eleventy.config.js` 中的关键函数

#### `module.exports = function(eleventyConfig)` — 主配置函数

Eleventy 配置入口，所有插件、过滤器、Transform 注册均在此完成。

#### `addPassthroughCopy` 配置

```javascript
eleventyConfig.addPassthroughCopy({
  "./public/": "/public/",                                          // 静态资源原样复制
  "./node_modules/prismjs/themes/prism-okaidia.css": "/css/prism-okaidia.css"  // Prism 主题
});
```

**作用**：将 `public/` 目录原样复制到 `_site/public/`，并将 Prism 主题 CSS 复制到 `_site/css/`。

#### `addWatchTarget`

```javascript
eleventyConfig.addWatchTarget("content/**/*.{svg,webp,png,jpeg}");
```

**作用**：监视内容目录下的图片文件变更，触发热重载。

#### Transform: `externalBlank`

```javascript
eleventyConfig.addTransform("externalBlank", (content, outputPath) => {
  if (!outputPath || !outputPath.endsWith(".html")) return content;
  return content.replace(/<a\s+([^>]*?)href="(https?:\/\/[^"]*?)"([^>]*)>/gi, ...);
});
```

**作用**：构建时遍历所有生成的 HTML，对指向**非本站域名**的 `<a>` 标签自动添加 `target="_blank" rel="noopener"`，避免在文章中手动维护。

**关键逻辑**：
- 仅处理 `.html` 输出
- 若 `<a>` 已有 `target=` 属性则跳过
- 通过 `new URL(url).host === siteHost` 判断是否本站内链

#### Transform: `alertBlocks`

```javascript
eleventyConfig.addTransform("alertBlocks", (content, outputPath) => { ... });
```

**作用**：将 Markdown 的 GFM 警报块语法转换为带样式的 HTML blockquote。

**支持的警报类型**：

| 语法 | 输出 |
|---|---|
| `> [!tip]` | `💡 提示：` |
| `> [!info]` | `📖 信息：` |
| `> [!warning]` | `⚠️ 注意：` |
| `> [!caution]` | `🚨 警告：` |

**输出 HTML 形式**：
```html
<blockquote data-alert data-alert-tip>
  <p>💡 提示： 内容...</p>
</blockquote>
```

#### Filter: `readableDate`

```javascript
eleventyConfig.addFilter("readableDate", (dateObj, format, zone) => {
  return DateTime.fromJSDate(dateObj, { zone: zone || "utc" })
    .setLocale("zh-CN")
    .toFormat(format || "yyyy年M月d日");
});
```

**作用**：将 Date 对象格式化为中文可读日期，默认格式 `2026年3月5日`。基于 `luxon` 库，时区默认 UTC。

#### Filter: `htmlDateString`

```javascript
eleventyConfig.addFilter('htmlDateString', (dateObj) => {
  return DateTime.fromJSDate(dateObj, {zone: 'utc'}).toFormat('yyyy-LL-dd');
});
```

**作用**：输出 ISO 风格日期 `2026-03-05`，常用于 `<time datetime="...">` 属性。

#### Filter: `sortByPin`（自定义）

```javascript
eleventyConfig.addFilter("sortByPin", (posts) => {
  return [...posts].sort((a, b) => {
    if (a.data.pin && !b.data.pin) return -1;
    if (!a.data.pin && b.data.pin) return 1;
    return b.date - a.date;
  });
});
```

**作用**：文章列表排序——置顶文章（`pin: true`）排最前，其余按日期倒序。**注意**：返回新数组，不修改原集合。

#### Filter: `head`

```javascript
eleventyConfig.addFilter("head", (array, n) => {
  if(!Array.isArray(array) || array.length === 0) return [];
  if( n < 0 ) return array.slice(n);
  return array.slice(0, n);
});
```

**作用**：取数组前 N 项（n 为负数时取后 N 项）。

#### Filter: `min`

```javascript
eleventyConfig.addFilter("min", (...numbers) => Math.min.apply(null, numbers));
```

**作用**：返回参数中的最小值。

#### Filter: `getAllTags`

```javascript
eleventyConfig.addFilter("getAllTags", collection => {
  let tagSet = new Set();
  for(let item of collection) {
    (item.data.tags || []).forEach(tag => tagSet.add(tag));
  }
  return Array.from(tagSet);
});
```

**作用**：从集合中收集所有出现过的标签，去重返回数组。用于标签列表页。

#### Filter: `filterTagList`

```javascript
eleventyConfig.addFilter("filterTagList", function filterTagList(tags) {
  return (tags || []).filter(tag => ["all", "nav", "post", "posts"].indexOf(tag) === -1);
});
```

**作用**：过滤掉系统保留标签（`all`、`nav`、`post`、`posts`），避免在标签云中显示内部标签。

#### Markdown 库增强

```javascript
eleventyConfig.amendLibrary("md", mdLib => {
  mdLib.set({ linkify: true });   // 自动识别 URL 文本为链接
  mdLib.use(markdownItAnchor, {   // 为 h1-h4 添加锚点
    permalink: markdownItAnchor.permalink.ariaHidden({
      placement: "after",
      class: "header-anchor",
      symbol: "#",
      ariaHidden: false,
    }),
    level: [1,2,3,4],
    slugify: eleventyConfig.getFilter("slugify")
  });
});
```

**作用**：
- 启用 `linkify`：自动将纯文本 URL 转为可点击链接
- 启用 `markdown-it-anchor`：为 H1-H4 标题自动添加 `#` 锚点链接，支持深链跳转

#### 配置返回对象

```javascript
return {
  templateFormats: ["md", "njk", "html", "liquid"],
  markdownTemplateEngine: "njk",   // Markdown 中可使用 Nunjucks 语法
  htmlTemplateEngine: "njk",       // HTML 中可使用 Nunjucks 语法
  dir: {
    input: "content",              // 输入根目录
    includes: "../_includes",      // 模板包含目录（相对 input）
    data: "../_data",              // 全局数据目录（相对 input）
    output: "_site"                // 输出目录
  },
  pathPrefix: "/",
};
```

### 6.2 `eleventy.config.drafts.js` 中的关键函数

#### `eleventyComputedPermalink()`

```javascript
function eleventyComputedPermalink() {
  return (data) => {
    if(data.draft && !process.env.BUILD_DRAFTS) {
      return false;  // 草稿在非 watch/serve 构建时不输出文件
    }
    return data.permalink;
  }
};
```

**作用**：计算属性的 permalink。草稿文章在 `--serve`/`--watch` 之外的构建中返回 `false`，Eleventy 会跳过该文件的输出。

**注意**：因 `addGlobalData` 顶层函数会被立即执行，需嵌套函数返回实际计算函数。

#### `eleventyComputedExcludeFromCollections()`

```javascript
function eleventyComputedExcludeFromCollections() {
  return (data) => {
    if(data.draft && !process.env.BUILD_DRAFTS) {
      return true;  // 草稿从集合中排除，避免出现在列表/RSS
    }
    return data.eleventyExcludeFromCollections;
  }
};
```

**作用**：草稿在生产构建时从 `collections` 中排除，确保不出现在文章列表、RSS、上下篇导航中。

#### `eleventy.before` 事件钩子

```javascript
eleventyConfig.on("eleventy.before", ({runMode}) => {
  let text = "Excluding";
  if(runMode === "serve" || runMode === "watch") {
    process.env.BUILD_DRAFTS = true;
    text = "Including";
  }
  if(!logged) {
    console.log(`[11ty/eleventy-base-blog] ${text} drafts.`);
    logged = true;
  }
});
```

**作用**：在每次构建开始前，根据运行模式设置 `BUILD_DRAFTS` 环境变量。`logged` 标志确保日志只打印一次。

### 6.3 `eleventy.config.images.js` 中的关键函数

#### `relativeToInputPath(inputPath, relativeFilePath)`

```javascript
function relativeToInputPath(inputPath, relativeFilePath) {
  let split = inputPath.split("/");
  split.pop();
  return path.resolve(split.join(path.sep), relativeFilePath);
}
```

**作用**：将文章中的相对图片路径解析为相对于文章文件所在目录的绝对路径。例如文章 `content/blog/ai-agent.md` 中的 `./cover.png` 会被解析为 `content/blog/cover.png`。

#### `isFullUrl(url)`

```javascript
function isFullUrl(url) {
  try { new URL(url); return true; } catch(e) { return false; }
}
```

**作用**：判断字符串是否为完整 URL（`http://` 或 `https://`）。用于区分远程图片与本地图片。

#### `imageShortcode(src, alt, widths, sizes)`

```javascript
eleventyConfig.addAsyncShortcode("image", async function imageShortcode(src, alt, widths, sizes) {
  let formats = ["avif", "webp", "auto"];
  let input = isFullUrl(src) ? src : relativeToInputPath(this.page.inputPath, src);
  let metadata = await eleventyImage(input, {
    widths: widths || ["auto"],
    formats,
    outputDir: path.join(eleventyConfig.dir.output, "img"),
  });
  let imageAttributes = { alt, sizes, loading: "lazy", decoding: "async" };
  return eleventyImage.generateHTML(metadata, imageAttributes);
});
```

**作用**：注册 `{% image %}` 异步短代码。

**使用方式**（在 Markdown 中）：
```nunjucks
{% image "./cover.png", "封面图", [300, 600], "(min-width: 30em) 50vw, 100vw" %}
```

**输出**：自动生成 `<picture>` 标签，含 AVIF/WebP 多源 `srcset`，并附 `width`/`height`/`loading="lazy"`/`decoding="async"` 属性。

### 6.4 模板中的关键代码片段

#### `base.njk` — 客户端图片灯箱 JS

```javascript
// 核心逻辑：监听 main 区域点击，若点击 img 则打开灯箱
document.querySelector('main').addEventListener('click', e => {
  const img = e.target.closest('img');
  if (!img || !img.closest('main')) return;
  if (img.closest('.postlist-cover-link') || img.closest('.site-icon')) return;
  lbImg.src = img.src;
  lb.classList.add('active');
  lbImg.addEventListener('load', resetView, { once: true });
});
```

**功能**：
- 点击正文图片放大显示
- 滚轮缩放（0.5x ~ 20x）
- 鼠标拖拽平移
- `Esc` 关闭、`R` 重置视图
- 自动适配窗口大小（`resetView` 计算 `scale`）

#### `search.njk` — 客户端搜索逻辑

```javascript
async function loadFeed() {
  const res = await fetch('/feed/feed.xml');
  const xml = await res.text();
  const doc = parser.parseFromString(xml, 'text/xml');
  // 解析所有 entry 为 { title, link, desc, cover, content }
  feedData = [...];
}

function doSearch() {
  // 按空格分词，统计 title/desc/content 中的匹配数
  // 按总匹配数倒序排序，渲染结果
}
```

**特点**：
- 无需服务端搜索接口，复用 `/feed/feed.xml` 作为搜索索引
- 客户端 DOMParser 解析 XML
- 多关键词匹配计数排序
- 200ms 防抖

---

## 7. 模板系统

### 7.1 布局层级

```
base.njk (基础 HTML 骨架)
  ├── home.njk (首页/列表页：仅透传 content)
  └── post.njk (文章详情页：标题/元数据/正文/Giscus/上下篇)
```

### 7.2 `_includes/layouts/base.njk` — 基础布局

**职责**：定义所有页面的 HTML 外壳。

**关键元素**：
- `<head>`：标题、描述、Favicon、Feed 链接、CSS 打包（`{% getBundle "css" %}`）
- `<header>`：站点图标 + 标题 + 导航菜单（基于 `eleventyNavigation`）
- `<main id="skip">`：页面主内容区（`{{ content | safe }}`）
- `<footer>`：页脚（当前为空）
- 灯箱 DOM 结构 + 灯箱交互 JS（`{% js %}...{% endjs %}`）
- JS 打包输出（`{% getBundle "js" %}`）

**导航生成逻辑**：
```nunjucks
{%- for entry in collections.all | eleventyNavigation %}
  <li class="nav-item">
    <a href="{{ entry.url }}"{% if entry.url == page.url %} aria-current="page"{% endif %}>
      {{ entry.title }}
    </a>
  </li>
{%- endfor %}
```

通过 Frontmatter 中的 `eleventyNavigation.key` 与 `order` 控制导航项顺序。

### 7.3 `_includes/layouts/home.njk` — 首页布局

```nunjucks
---
layout: layouts/base.njk
---
{{ content | safe }}
```

**职责**：极简透传布局，仅继承 base.njk 并输出 content。首页、标签列表页、404 页均使用此布局。

### 7.4 `_includes/layouts/post.njk` — 文章布局

**职责**：渲染单篇文章详情页。

**关键结构**：
- `<h1>{{ title }}</h1>` — 文章标题
- `<blockquote class="post-description">` — 描述（若存在）
- 元数据列表：📅 日期 + 标签链接
- `{{ content | safe }}` — 正文
- Giscus 评论脚本（基于 GitHub Discussions，仓库 `afoim/giscus-fuwari`）
- 上下篇导航（`collections.posts | getPreviousCollectionItem` / `getNextCollectionItem`）

### 7.5 `_includes/postslist.njk` — 文章列表组件

**职责**：可复用的文章列表渲染组件。

**特性**：
- 通过 `sortByPin` 过滤器置顶文章排前
- 置顶文章标题带 `pin-gradient` 渐变动画
- 显示封面图（`post.data.coverImage`）
- 显示描述与日期

**使用方式**：
```nunjucks
{% set postslist = collections.posts %}
{% include "postslist.njk" %}
```

### 7.6 内容页面一览

| 文件 | 输出路径 | 布局 | 用途 |
|---|---|---|---|
| `content/index.njk` | `/` | home.njk | 首页（显示文章列表） |
| `content/about/index.md` | `/about/` | base.njk | 关于页 |
| `content/posts.njk` | `/posts/` | （无布局，HTML 重定向） | 跳转回首页 |
| `content/tags-list.njk` | `/tags/` | home.njk | 所有标签列表 |
| `content/tags.njk` | `/tags/{tag}/` | home.njk | 单标签下的文章列表（分页生成） |
| `content/search.njk` | `/search/` | base.njk | 搜索页 |
| `content/404.md` | `/404.html` | home.njk | 404 页面 |
| `content/blog/*.md` | `/posts/{fileSlug}/` | post.njk | 博客文章详情页 |
| `content/feed/feed.njk` | `/feed/feed.xml` | （无布局） | Atom Feed |
| `content/feed/json.njk` | `/feed/feed.json` | （无布局） | JSON Feed |
| `content/sitemap/sitemap.xml.njk` | `/sitemap.xml` | （无布局） | 站点地图 |

### 7.7 标签页分页机制（`content/tags.njk`）

```yaml
---
pagination:
  data: collections
  size: 1
  alias: tag
  filter:
    - all
    - post
    - posts
    - tagList
  addAllPagesToCollections: true
eleventyComputed:
  title: Tagged "{{ tag }}"
permalink: /tags/{{ tag | slugify }}/
---
```

**机制**：通过 Eleventy 分页功能，遍历所有 `collections`（每个 tag 对应一个 collection），每个 tag 生成一个独立页面。`filter` 排除系统保留集合。

---

## 8. 数据流与构建流程

### 8.1 数据级联（Data Cascade）

Eleventy 数据按以下优先级合并（高优先级覆盖低优先级）：

```
1. 全局数据 (_data/*.js)               [最低]
2. 目录数据 (content/blog/blog.11tydata.js)
3. 模板 Frontmatter (--- ... ---)      [最高]
```

**示例**：一篇 `content/blog/ai-agent.md` 文章的最终数据：

| 字段 | 来源 | 值 |
|---|---|---|
| `tags` | blog.11tydata.js | `["posts"]` |
| `layout` | blog.11tydata.js | `layouts/post.njk` |
| `permalink` | blog.11tydata.js (eleventyComputed) | `/posts/ai-agent/` |
| `title` | Frontmatter | `AI Agent 二三事` |
| `date` | Frontmatter | `2026-03-05` |
| `description` | Frontmatter | `从实战角度对比...` |
| `draft` | Frontmatter | `false` |
| `coverImage` | Frontmatter | `/public/img/ai-agent-ai-agent.png` |
| `metadata.*` | _data/metadata.js | 站点元数据 |

### 8.2 构建流程

```
[启动构建]
    │
    ▼
[触发 eleventy.before 钩子]
    │  - drafts 插件根据 runMode 设置 BUILD_DRAFTS
    ▼
[读取配置 eleventy.config.js]
    │  - 注册插件、过滤器、Transform
    ▼
[扫描 content/ 目录]
    │  - 识别 .md / .njk / .html / .liquid 模板
    ▼
[数据级联合并]
    │  - 全局数据 + 目录数据 + Frontmatter
    ▼
[草稿过滤]
    │  - draft && !BUILD_DRAFTS → permalink=false，跳过输出
    ▼
[模板渲染]
    │  - Markdown → HTML（含 linkify、anchor）
    │  - Nunjucks → HTML
    │  - 应用布局链（post.njk → base.njk）
    ▼
[Transform 处理]
    │  - externalBlank: 外链加 target=_blank
    │  - alertBlocks: GFM 警报块转 HTML
    ▼
[图片优化]
    │  - {% image %} 短代码生成 AVIF/WebP
    ▼
[写入 _site/]
    │  - HTML / CSS / img / feed.xml / sitemap.xml
    ▼
[完成]
```

### 8.3 集合（Collections）

项目使用的关键集合：

| 集合名 | 来源 | 用途 |
|---|---|---|
| `all` | 所有未排除的模板 | 导航、sitemap |
| `posts` | `tags: ["posts"]`（来自 blog.11tydata.js） | 文章列表、Feed、上下篇 |
| `{tag}` | 自动按 tag 派生 | 标签页 |

---

## 9. 依赖关系

### 9.1 运行时依赖（devDependencies）

| 依赖 | 版本 | 用途 |
|---|---|---|
| `@11ty/eleventy` | ^2.0.1 | 静态站点生成器核心 |
| `@11ty/eleventy-img` | ^3.1.1 | 图片优化短代码 |
| `@11ty/eleventy-navigation` | ^0.3.5 | 导航菜单插件 |
| `@11ty/eleventy-plugin-bundle` | ^1.0.4 | CSS/JS 打包插件 |
| `@11ty/eleventy-plugin-rss` | ^1.2.0 | RSS 过滤器（`dateToRfc3339`、`htmlBaseUrl`） |
| `@11ty/eleventy-plugin-syntaxhighlight` | ^5.0.0 | 代码语法高亮（Prism） |
| `luxon` | ^3.3.0 | 日期处理库（`DateTime`） |
| `markdown-it-anchor` | ^8.6.7 | Markdown 标题锚点 |

### 9.2 依赖关系图

```
@11ty/eleventy (核心)
  ├── @11ty/eleventy-navigation        → 导航菜单
  ├── @11ty/eleventy-plugin-bundle      → CSS/JS 内联打包
  ├── @11ty/eleventy-plugin-rss         → RSS 过滤器
  ├── @11ty/eleventy-plugin-syntaxhighlight → 代码高亮
  ├── @11ty/eleventy-img                → 图片优化（被 eleventy.config.images.js 使用）
  ├── luxon                              → 日期格式化（被 readableDate/htmlDateString 使用）
  └── markdown-it-anchor                 → 标题锚点（amendLibrary 增强 md 库）
```

### 9.3 外部服务依赖

| 服务 | 用途 | 引用位置 |
|---|---|---|
| Giscus (`giscus.app`) | 评论区 | `post.njk` |
| GitHub Discussions | 评论存储（仓库 `afoim/giscus-fuwari`） | `post.njk` |
| QQ 头像 (`q2.qlogo.cn`) | 站点 Favicon | `_data/metadata.js` |
| PagesCMS | 在线内容编辑 | `.pages.yml` |
| Netlify Lighthouse | 性能监控 | `netlify.toml` |
| Cloudflare Workers | 部署目标 | `.pages.yml` deploy action |

### 9.4 模块间依赖

```
eleventy.config.js
  ├── 依赖 _data/metadata.js       (读取 siteUrl 用于 externalBlank)
  ├── 依赖 eleventy.config.drafts.js (插件)
  ├── 依赖 eleventy.config.images.js (插件)
  └── 依赖 luxon, markdown-it-anchor

content/blog/*.md
  └── 依赖 blog.11tydata.js         (默认 layout/tags/permalink)
       └── 依赖 layouts/post.njk
            └── 依赖 layouts/base.njk
                 └── 依赖 _data/metadata.js (站点标题/图标/导航)
                 └── 依赖 public/css/index.css (样式)
                 └── 依赖 collections.all | eleventyNavigation (导航)

content/feed/feed.njk
  └── 依赖 @11ty/eleventy-plugin-rss 过滤器
       (dateToRfc3339, htmlBaseUrl, transformWithHtmlBase)

content/search.njk
  └── 运行时 fetch /feed/feed.xml   (客户端搜索索引源)
```

---

## 10. 项目运行方式

### 10.1 环境要求

- **Node.js**：>= 14（推荐 16，见 `.nvmrc`）
- **npm**：随 Node 安装
- **Git**：用于克隆仓库

### 10.2 安装

```bash
# 1. 克隆仓库
git clone <仓库地址>
cd <项目目录>

# 2. （可选）使用 nvm 切换 Node 版本
nvm use    # 自动读取 .nvmrc，切换到 Node 16

# 3. 安装依赖
npm install
```

> **注意**：`package-lock.json` 被 `.gitignore` 忽略，依赖版本可能浮动。如遇兼容问题，可参考 `package.json` 中的 `^` 版本范围锁定。

### 10.3 本地开发

```bash
# 启动本地开发服务器（含热重载，默认 http://localhost:8080）
npm start

# 等价于
npx @11ty/eleventy --serve --quiet
```

**开发模式特性**：
- 自动监视文件变更并热重载
- `draft: true` 的文章**可见**（控制台输出 `[11ty/eleventy-base-blog] Including drafts.`）
- 监视 `content/**/*.{svg,webp,png,jpeg}` 图片变更

### 10.4 生产构建

```bash
# 生成生产构建到 _site/ 目录
npm run build

# 等价于
npx @11ty/eleventy
```

**生产模式特性**：
- `draft: true` 的文章**不输出**且从集合中排除
- 输出纯净静态文件到 `_site/`

### 10.5 调试与基准测试

```bash
# 调试模式（输出详细 Eleventy 内部日志）
npm run debug

# 调试 + 本地服务器
npm run debugstart

# 性能基准测试
npm run benchmark
```

### 10.6 GitHub Pages 子路径构建

若部署到 GitHub Pages 的项目子路径（如 `https://user.github.io/repo/`）：

```bash
npm run build-ghpages
# 使用 --pathprefix=/eleventy-base-blog/，需替换为你的仓库名
```

### 10.7 本地预览构建产物

构建完成后，可用任意静态服务器预览：

```bash
npx serve _site
# 或
npx http-server _site -p 8080
```

---

## 11. 部署方式

### 11.1 Netlify 部署

项目已配置 `netlify.toml`：

1. 在 Netlify 后台连接 GitHub 仓库
2. 自动识别配置：
   - 构建命令：`npm run build`
   - 发布目录：`_site`
3. 自动启用 Lighthouse 插件，要求四项指标均达到 1.0

### 11.2 Cloudflare Pages / Workers 部署

通过 `.pages.yml` 中的 deploy action 配置：
- 工作流：`deploy.yml`
- 触发方式：在 PagesCMS 界面点击 "Deploy to Cloudflare"
- 需要在 `.github/workflows/deploy.yml` 中定义具体部署工作流（本项目未包含该文件，需自建）

### 11.3 GitHub Pages 部署

1. 运行 `npm run build-ghpages`（替换 pathprefix 为你的仓库名）
2. 将 `_site/` 推送到 `gh-pages` 分支或通过 GitHub Actions 部署
3. 项目根目录的 `.nojekyll` 文件确保 GitHub Pages 不使用 Jekyll 处理

### 11.4 其他静态托管

由于输出为纯静态文件，可部署到任意静态托管服务：
- Vercel
- Cloudflare Pages
- 阿里云 OSS / 腾讯云 COS
- 任意 Nginx/Apache 服务器

只需将 `_site/` 目录内容上传即可。

---

## 12. 内容创作指南

### 12.1 创建新文章

在 `content/blog/` 下新建 `.md` 文件，文件名即为 URL slug（如 `my-post.md` → `/posts/my-post/`）。

### 12.2 Frontmatter 字段

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `title` | string | 是 | — | 文章标题 |
| `date` | string/date | 是 | — | 发布日期（如 `'2026-03-05'`） |
| `description` | string | 否 | — | 文章描述（用于列表、SEO、Feed） |
| `draft` | boolean | 否 | `false` | 是否为草稿（仅开发模式可见） |
| `pin` | boolean | 否 | `false` | 是否置顶（列表页置顶 + 渐变动画） |
| `tags` | string[] | 否 | `["posts"]`（来自 11tydata） | 标签数组 |
| `coverImage` | string | 否 | — | 封面图路径（如 `/public/img/xxx.png`） |
| `layout` | string | 否 | `layouts/post.njk` | 布局模板 |
| `permalink` | string | 否 | `/posts/{fileSlug}/` | 自定义永久链接 |
| `eleventyNavigation` | object | 否 | — | 导航菜单配置（key/order） |
| `eleventyExcludeFromCollections` | boolean | 否 | `false` | 是否从集合中排除 |

### 12.3 Frontmatter 示例

**最小化文章**：
```yaml
---
title: 我的第一篇文章
date: '2026-07-05'
---
正文内容...
```

**完整配置文章**：
```yaml
---
title: AI Agent 二三事
date: '2026-03-05'
description: 从实战角度对比 AI IDE 与 AI Agent，并演示如何用 ZCF 快速安装和配置 Codex。
draft: false
tags: []
coverImage: /public/img/ai-agent-ai-agent.png
---
正文内容...
```

**置顶文章**：
```yaml
---
title: 加群向导
date: '2025-05-23'
description: 关于如何联系二叉树树~
pin: true
draft: false
coverImage: /public/img/pin-cute.webp
---
```

### 12.4 Markdown 扩展语法

#### 自动链接
```markdown
访问 https://feed.2x.nz/  （自动转为可点击链接）
```

#### 标题锚点
所有 H1-H4 标题自动添加 `#` 锚点，支持深链跳转。

#### GFM 警报块
```markdown
> [!tip] 这是一个提示
> 内容...

> [!warning] 这是一个警告
> 内容...
```

#### 图片引用
```markdown
![替代文本](/public/img/xxx.png)

<!-- 推荐使用 image 短代码（自动优化） -->
{% image "./cover.png", "封面图", [300, 600], "(min-width: 30em) 50vw, 100vw" %}
```

#### 代码高亮
````markdown
```javascript
function hello() {
  console.log("Hello");
}
```
````

### 12.5 图片管理

- **文章配图**：放置于 `public/img/`，命名建议 `{文章slug}-{序号}.{webp|png}`
- **封面图**：通过 Frontmatter `coverImage` 字段指定
- **图片优化**：使用 `{% image %}` 短代码自动生成 AVIF/WebP 多格式
- **灯箱**：正文中的图片点击后自动放大（无需额外配置）

### 12.6 通过 PagesCMS 在线编辑

1. 访问 [PagesCMS](https://pagescms.org) 并连接 GitHub 仓库
2. 读取 `.pages.yml` 配置，提供可视化编辑界面
3. 可编辑：文章、关于页、元数据
4. 可触发部署与文章广播工作流

---

## 13. 定制与扩展点

### 13.1 修改站点信息

编辑 `_data/metadata.js` 修改标题、URL、作者、描述、图标。

### 13.2 修改样式

编辑 `public/css/index.css`。关键 CSS 变量：

```css
:root {
  --font-family: -apple-system, system-ui, sans-serif;
  --background-color: #15202b;          /* 暗色背景 */
  --text-color: var(--color-gray-90);   /* 正文颜色 */
  --text-color-link: #1493fb;           /* 链接颜色 */
}
```

### 13.3 添加自定义 Filter

在 `eleventy.config.js` 中添加：

```javascript
eleventyConfig.addFilter("myFilter", (value) => { ... });
```

### 13.4 添加自定义 Transform

```javascript
eleventyConfig.addTransform("myTransform", (content, outputPath) => {
  if (!outputPath || !outputPath.endsWith(".html")) return content;
  // 处理 content...
  return content;
});
```

### 13.5 修改文章 URL 结构

编辑 `content/blog/blog.11tydata.js`：

```javascript
eleventyComputed: {
  permalink: data => `/blog/${data.page.fileSlug}/`  // 改为 /blog/xxx/
}
```

### 13.6 添加导航菜单项

在页面的 Frontmatter 中添加：

```yaml
eleventyNavigation:
  key: 自定义页
  order: 5
```

### 13.7 修改评论区配置

编辑 `_includes/layouts/post.njk` 中的 Giscus 配置：
- `data-repo`：GitHub 仓库
- `data-repo-id`：仓库 ID
- `data-category`：Discussion 分类
- `data-theme`：主题（dark/light）

### 13.8 添加新页面

在 `content/` 下新建 `.md` 或 `.njk` 文件，按需添加 Frontmatter：

```yaml
---
layout: layouts/base.njk
eleventyNavigation:
  key: 新页面
  order: 6
permalink: /new-page/
---
# 新页面内容
```

---

## 附录：关键文件速查表

| 任务 | 文件 |
|---|---|
| 修改站点标题/作者 | `_data/metadata.js` |
| 修改构建配置 | `eleventy.config.js` |
| 修改文章默认 layout/permalink | `content/blog/blog.11tydata.js` |
| 修改草稿行为 | `eleventy.config.drafts.js` |
| 修改图片短代码 | `eleventy.config.images.js` |
| 修改全站样式 | `public/css/index.css` |
| 修改基础 HTML 结构 | `_includes/layouts/base.njk` |
| 修改文章页布局 | `_includes/layouts/post.njk` |
| 修改文章列表渲染 | `_includes/postslist.njk` |
| 修改评论区 | `_includes/layouts/post.njk`（Giscus 配置） |
| 修改搜索功能 | `content/search.njk` |
| 修改 RSS 输出 | `content/feed/feed.njk` |
| 修改 sitemap | `content/sitemap/sitemap.xml.njk` |
| 修改导航菜单 | 各页面的 `eleventyNavigation` Frontmatter |
| 修改部署配置 | `netlify.toml` / `.pages.yml` |
| 修改 Node 版本 | `.nvmrc` |

---

*本文档基于项目代码现状生成，最后更新时间：2026-07-05*
