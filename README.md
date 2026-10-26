# AI 应用工程师技术博客

这是一个基于 Hugo 与 Stack 主题构建的个人技术博客，主要记录 RAG、Agent、LLM 评测、模型应用工程和算法练习。

## 本地运行

环境要求：

- Hugo Extended 0.162.1
- Go（用于解析 Hugo Module 主题依赖）

```bash
hugo server -D
```

默认访问 `http://localhost:1313/`。生产构建命令：

```bash
hugo --cleanDestinationDir --minify --printI18nWarnings --printPathWarnings
```

## 内容结构

- `content/projects/`：项目案例，强调问题、架构、取舍、验证边界与复盘。
- `content/posts/ai_notes/`：AI 应用工程技术文章。
- `content/leetcode/`：算法题解，使用独立 section，但保留历史 URL。
- `content/tools/`：可直接在浏览器使用的小工具。
- `content/about/`：个人能力与公开联系入口。

## 内容规范

公开内容遵循以下原则：

1. 不保留 `【待补充】`、虚假链接或未经验证的指标。
2. 性能数字必须同时说明数据集、模型版本、运行环境和评测方法。
3. 项目页区分已验证实现、设计方案和后续工作。
4. 文章 front matter 至少包含 `date`、`draft`、`title`、`tags`、`categories` 和 `description`。
5. 内部链接优先使用站点绝对路径，例如 `/projects/deep_search/`。

## 主题与自定义

主题通过 Hugo Module 引入：`github.com/CaiJimmy/hugo-theme-stack/v3`。

主要自定义文件：

- `hugo.yaml`：导航、站点元数据、主题参数和内容分区。
- `layouts/index.html`：求职首页、精选内容和最新文章。
- `layouts/partials/article-list/homepage.html`：首页卡片。
- `assets/scss/custom.scss`：站点视觉、响应式和可访问性样式。
- `layouts/page/search.json`：搜索索引数据。

## 部署与质量检查

推送到 `main` 后，`.github/workflows/hugo.yaml` 会：

1. 检查公开 Markdown 中是否残留占位符；
2. 使用固定版本 Hugo 执行生产构建；
3. 验证首页、项目、搜索、分类、robots.txt 和 sitemap 等关键产物；
4. 检查生成 HTML 的内部链接；
5. 部署至 GitHub Pages。

`public/`、`resources/` 和 `.qoder/` 均为生成或本地目录，已通过 `.gitignore` 排除，不作为源代码维护。
