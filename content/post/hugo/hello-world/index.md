---
title: HUGO搭建笔记
description: Welcome to Hugo Theme Stack
slug: Note
date: 2022-03-06 00:00:00+0000
image: cover.jpg
categories:
    - Example Category
tags:
    - Example Tag
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
### HUGO搭建笔记

#### 通过Git推送到GitHub

1. 安装git，下一步点到底
https://git-scm.com/install/windows

2. 把 GitHub 仓库克隆到本地
```
    git clone https://github.com/AX7Princess/AX7Princess.github.io.git
    cd AX7Princess.github.io
```

3. 在本地写文章
   - 用 VS Code 打开这个文件夹。
   - 在 content/posts/ 目录下新建 .md 文件，用 Markdown 写文章。
   - 如果需要图片，放到 static/images/ 目录下。
<br>
4. 推送到 GitHub

在本地终端（项目目录下）执行：
```
git add .
git commit -m "新增文章：xxx"
git push
```
你的代码就会推送到 GitHub。

5. 服务器自动拉取并构建

推送后，你需要让服务器上的 /root/my-blog 目录自动拉取最新代码并重新生成网站。

在服务器上执行：
```
crontab -e
```
执行后会让打开编辑器，这里选nano，在文件末尾添加一行（每 5 分钟检查一次更新）：
```
*/5 * * * * cd /root/my-blog && git pull && hugo --minify
```
这行命令的含义：

- */5 * * * *：每5分钟执行一次。

- cd /root/my-blog：进入博客目录。

- git pull：从 GitHub 拉取最新代码。

- hugo --minify：重新生成静态网站。

验证定时任务是否添加成功
```
crontab -l
```
会显示你刚才添加的那一行，说明已生效。

Welcome to Hugo theme Stack. This is your first post. Edit or delete it, then start writing!

For more information about this theme, check the documentation: https://stack.jimmycai.com/

Want a site like this? Check out [hugo-theme-stack-stater](https://github.com/CaiJimmy/hugo-theme-stack-starter)

> Photo by [Pawel Czerwinski](https://unsplash.com/@pawel_czerwinski) on [Unsplash](https://unsplash.com/)