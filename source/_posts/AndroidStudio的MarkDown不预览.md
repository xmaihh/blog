---
title: AndroidStudio的MarkDown不预览
date: 2022-06-23 17:27:53
categories: Android
tags: [AndroidStudio,Android]
toc: false
description: 苦しみは 成長から生まれる 副産物。 それだけで 終わってしまうことは ありません。
---

Markdown Editor 插件后提示`Your environment does not support JCEF, cannot use Markdown Editor`

# 解决：

在idea中按快捷键`CTRL+SHIFT+A`,打开动作指令窗口, 或者双击Shift在最上方选择Action, 在输入框中输入`Choose Boot Java Runtime for the IDE`(其实输入个Choose Boot就提示出来了), 在弹出窗口中的select中选择一个 xxx JetBrains Runtime With JCEF, 点OK等待下载完成重启即可。

<img src="https://s2.loli.net/2022/06/23/pkS7o5gTmelwO3A.png" alt="image-20220623174608039" style="zoom:50%;" />
