---
title: "把博客塞进 GitHub Pages，踩的四个坑"
date: 2026-10-27
draft: false
tags: ["Hugo", "GitHub"]
categories: ["踩坑记录"]
ShowToc: true
---

站点是 Hugo + PaperMod，托管在 GitHub Pages。听起来像十分钟的活，我折腾了小半个晚上。

## 坑一：hugo 命令找不到

Windows 上用 winget 装完 Hugo Extended，终端里敲 `hugo` 还是"无法将 hugo 项识别为 cmdlet"。

因为 PATH 只在装完之后新开的终端里生效。我这个终端是之前开的，得关掉重开。我没关，直接用完整路径调用：

```powershell
C:\Users\<我>\AppData\Local\Microsoft\WinGet\Packages\Hugo.Hugo.Extended_...\hugo.exe
```

难看，但能用。后来重开终端，`hugo` 就正常了。

## 坑二：在错的目录里 git init

`hugo new site blog` 失败之后，我在 `C:\dev` 这个上层目录里执行了 `git init`，想"先建个仓库再说"。结果 `C:\dev` 变成了一个 git 仓库，`blog` 在里面只是个未跟踪的目录，后面一路全是麻烦。

删掉重来：`C:\dev\.git`、`.gitmodules`、`themes`、`blog` 全清掉，重新 `hugo new site blog`。

教训很直白：`git init` 之前先看一眼自己在哪。仓库范围一旦错了，改起来比重来还久。

## 坑三：主题拉不下来

PaperMod 官方推荐 `git clone` 到 `themes/`，我 clone 了三次全部中断，有一次卡在 30%。

换成下压缩包：

```powershell
Invoke-WebRequest -Uri "https://codeload.github.com/adityatelange/hugo-PaperMod/zip/refs/heads/master" -OutFile papermod.zip
Expand-Archive papermod.zip -DestinationPath themes
```

解出来的目录名要改成 `PaperMod`，和 `hugo.toml` 里的 `theme` 对上。

顺带记一个坑：用 `git submodule` 装主题，如果 `.gitmodules` 没跟着提交，GitHub 上构建的时候主题目录是空的，构建直接失败。我改用压缩包就是为了绕开这件事。

## 坑四：remote 地址里留着"你的用户名"

最蠢的一个。我把命令模板原样抄进了终端：

```bash
git remote add origin git@github.com:你的用户名/XiaoZege.github.io.git
```

`你的用户名` 这四个字真的被当成了用户名。于是 `git push` 一直报 `Repository not found`，我以为是没权限，以为是 SSH key 没配好，来回折腾了很久。

定位靠一条命令：

```bash
git remote -v
```

它把地址原样打出来，一眼就看见那个中文占位符。改掉：

```bash
git remote set-url origin git@github.com:XiaoZege/XiaoZege.github.io.git
```

## 还有一件把我卡住的事

推上去之后想去仓库里看 workflow 跑得怎么样，怎么也找不到——因为我一直在 Settings 里翻。

workflow 在 **Actions** 标签里。而且 GitHub Pages 要先把 Settings → Pages 里的 Source 设成 "GitHub Actions"，那个 workflow 才会真的跑起来。

顺序是：先开 Pages，再推代码，然后去 Actions 看结果。我一开始顺序搞反了，白等了一会儿。

## 现在的样子

https://XiaoZege.github.io

以后往 `content/posts/` 扔一个 markdown，`git push` 一下，一两分钟后自动更新。
