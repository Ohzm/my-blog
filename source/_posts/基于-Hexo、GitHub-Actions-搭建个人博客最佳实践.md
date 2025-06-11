---
title: 基于 Hexo、GitHub Actions 搭建个人博客最佳实践
date: 2025-06-11 09:18:40
tags: [Hexo, GitHub Actions, 搭建博客, 自动部署]
---

### GitHub 仓库设置

#### my-blog 仓库（Hexo 源码仓库）

- 放置你的 Markdown 文章、主题配置等
- 设置 GitHub Actions 自动生成博客并部署

#### ohzm.github.io 仓库（Pages 静态站点仓库）

- 仅托管 Hexo 构建生成的 public/ 目录的内容
- 不应包含源码、\_config.yml、node_modules 等
- 设置：

  - 打开仓库 → Settings → Pages
  - Source: Deploy from a branch
  - Branch: main，Folder: / (root)

### 1. 进入博客目录

cd c:/my-blog

### 2. 新建一篇博客文章

npx hexo n "我的第一篇博客"

成功后会创建一个文件，如：c:/my-blog/source/\_posts/我的第一篇博客.md

### 3. 编写博客内容

打开该 .md 文件，例如用 VS Code：
code source/\_posts/我的第一篇博客.md

在其中填写 Markdown 内容后，保存关闭

### 4. 预览博客效果（可选）

npx hexo clean

npx hexo g

npx hexo s

浏览器访问：http://localhost:4000 查看效果

### 5. 添加变更到 Git 并提交 (my-blog)

git add .

git commit -m "新增：第一篇博客文章"

git push orign main

自动触发 GitHub Actions（部署 public 到 ohzm.github.io

### 7. 检查部署是否成功

打开 my-blog 仓库的 Actions 页面：

- 查看是否触发了部署工作流

- 确认是否成功运行 hexo g + 部署到 yourname.github.io

### 8. 查看线上博客

打开浏览器访问：
https://yourname.github.io

你刚写的博客文章应该已经上线啦

### 安装主题(NexT)

1. 在 Hexo 博客根目录下执行：

   cd c:\my-blog

2. 使用 submodule 方式添加：

   git submodule add https://github.com/theme-next/hexo-theme-next themes/next

3. \_config.yml 修改主题配置：

   theme: next

4. 安装 NexT 推荐依赖（可选）

   这些依赖增强主题体验，如搜索、标签页支持等：

   npm install hexo-renderer-pug hexo-renderer-stylus --save

5. 本地预览确认主题已生效
   运行以下命令，确认主题加载成功：

   npx hexo clean

   npx hexo generate

   npx hexo server

   打开浏览器访问 http://localhost:4000，应该看到 NexT 的默认样式界面
