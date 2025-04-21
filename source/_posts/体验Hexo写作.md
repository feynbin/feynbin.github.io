---
title: 体验Hexo写作
top_img: false
abbrlink: 5b029f4f
date: 2025-04-21 23:48:54
updated:
tags:
categories:
keywords:
description:
cover:
highlight_shrink:
---

## **1. 本地写作与实时预览**

### **步骤说明**

1. **创建新文章**（无需全局安装 Hexo）：
   在项目根目录下运行：

   ```
   bunx hexo new "记一次 Hexo 的体验"
   ```

   - 文件会生成在 `source/_posts/记一次 Hexo 的体验.md`。
   - 使用 Typora/VSCode 编辑内容，保留自动生成的 Front Matter（如 `title`、`date`）。

2. **实时预览**：
   启动本地服务器（通过 Bun 运行）：

   ```
   bun run server  # 对应 package.json 中的脚本 "server": "hexo server"
   ```

   - 访问 `http://localhost:4000` 查看效果。
   - **修改内容后保存，浏览器会自动刷新**（Hexo 默认支持热更新）。

------

## **2. 推送到 GitHub Pages（仅文章内容）**

### **生成静态文件并部署**

1. **清理旧缓存并生成新文件**：

   ```
   bunx hexo clean && bunx hexo generate
   ```

   - 生成的静态文件在 `public/` 目录下。
   - **注意**：首次发布时，`public/` 文件夹可能不包含图片等资源（需后续访问触发缓存）。

2. **部署到 GitHub Pages 分支**（如 `gh-pages` 或 `page`）：

   ```
   bunx hexo deploy
   ```

   - 前提：在 `_config.yml` 中配置了正确的部署分支：

     ```
     deploy:
       type: git
       repo: git@github.com:你的用户名/你的仓库.git
       branch: page  # 指定分支名
     ```

------

## **3. 备份 Hexo 源码到 GitHub（防止数据丢失）**

### **将源码推送到 `main` 分支**

1. **初始化 Git 仓库**（如果尚未初始化）：

   ```
   git init
   git add .
   git commit -m "备份 Hexo 源码"
   ```

2. **关联远程仓库并推送**：

   ```
   git remote add origin git@github.com:你的用户名/你的仓库.git
   git branch -M main  # 确保分支名是 main
   git push -u origin main
   ```

   - **关键点**：
     - Hexo 源码（Markdown、主题、配置）保存在 `main` 分支。
     - 生成的静态文件（`public/`）通过 `hexo deploy` 推送到 `page` 分支。
     - 两者分离，互不干扰。

------

## **4. 注意事项**

### **缓存与资源管理**

- **图片缓存问题**：
  首次部署后，访问文章页面时，Hexo 会动态生成资源路径。确保：

  - 图片放在 `source/_posts/记一次 Hexo 的体验/` 文件夹内。
  - 引用时使用相对路径：`![描述](记一次 Hexo 的体验/photo.jpg)`。

- **强制更新缓存**：
  若 GitHub Pages 未显示最新内容，在部署时添加 `--force`：

  ```
  bunx hexo deploy --force
  ```

### **Bun 环境适配**

- 所有 `hexo` 命令均通过 `bunx` 运行（等价于 `npx`）。

- 若遇到权限问题，检查 `package.json` 中是否包含 Hexo 相关依赖：

  ```
  "dependencies": {
    "hexo": "^6.3.0",
    "hexo-deployer-git": "^3.0.0"
  }
  ```

------

## **5. 总结流程**

**这就是你未来向朋友介绍的完整流程！**

- 享受 Bun 的快速执行，无需全局安装 Hexo。
- 通过分支隔离源码和静态页面，安全备份。
- 实时预览和部署的流畅体验，正是 Hexo 的魅力所在。