---
layout:     post   				    # 使用的布局（不需要改）
title:      IDEA在MacOS 系统下，使用CMD+SHIFT+A 总是弹出 Terminal apropos问题 			# 标题 
subtitle:   IDEA,Terminal,apropos,cmd+shift+a,macos #副标题
date:       2019-05-21 				# 时间
author:     soulmz 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - IDEA
    - Terminal apropos
---

# IDEA在MacOS 系统下，使用CMD+SHIFT+A 总是弹出 Terminal apropos问题

> 系统: MacOS 10.14.4
>
> IDEA版本: 2018.2.7 IDEA

近期使用IDEA，写代码过程中，由于经常使用到 `cmd+shift+a` 来搜索。

总是会弹出如下窗口(截图):

![terminal](https://intellij-support.jetbrains.com/hc/article_attachments/360002207640/Region_capture_1.png)

## 原因:

macOS 10.14.4为终端中的搜索人页面索引添加新的默认快捷方式：

![masos](https://intellij-support.jetbrains.com/hc/article_attachments/360002207660/Screenshot_2019-05-08_at_11.21.08.png)

## 解决方式:

禁用掉 `Search man Page Index in Terminal` 或者 换个快捷键。

参考资料:

[ideallij-support](https://intellij-support.jetbrains.com/hc/en-us/articles/360005137400-Cmd-Shift-A-hotkey-opens-Terminal-with-apropos-search-instead-of-the-Find-Action-dialog)
