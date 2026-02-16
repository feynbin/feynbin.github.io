---
title: {{ title }}
date: {{ date }}
updated:
tags:
categories:
keywords:
description:
top_img: false
cover:
highlight_shrink:
---

## **一号标题**

### **二号标题**

**加粗**

博客目录运行：

```
hexo new "newpost"
```

- 文件会生成在 `source/_posts/newpost.md`。
- 使用 Typora/VSCode 编辑内容，保留自动生成的 Front Matter（如 `title`、`date`）。

1. **实时预览**：
   启动本地服务器:

   ```
   npm run server  # 对应 package.json 中的脚本 "server": "hexo server"
   ```

   - 访问 `http://localhost:4000` 查看效果。
   - **修改内容后保存，浏览器会自动刷新**（Hexo 默认支持热更新）。

------

## ** 推送到 GitHub Actions**

```
git add "本次修改的文件" //一般就是本次新添加的文章
git commit -m "new post"
git push origin
//github actions会自动编译博客到https://feynbin.github.io
```

