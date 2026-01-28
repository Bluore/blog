---
title: 模仿赛机 - abc题目快捷提交
published: 2026-01-28
description: ''
image: ''
tags: [开源]
category: '分享'
draft: false
lang: ''
---

关于`submit`的原理和使用

# 实现原理

使用时执行这个命令

```cmd
submit abc123d.cpp
```

会创建一个本地的服务器，监听`9974`端口，浏览器插件通过http轮训`localhost:9974`来获取提交的文件（最多监听10s）

浏览器插件可以通过文件名推断atcoder的对应题目的提交链接（使用cph的短名称自动命名），模拟输入代码和点击提交按钮（默认等待5s用来过cloudflare的人机验证）

:::warning
由于chorme对于插件有一定的限制，会在后台10s内杀掉插件，所以需要点击插件进入监听页面
:::

# 下载和使用

将编译好的exe程序添加到环境变量PATH中

:::tip
下载方式：[Github](https://github.com/bluore/submit)
::github{repo="Bluore/submit"}
:::

浏览器安装插件，使用时点击插件图标进入监听页面

:::tip
下载方式：[Github](https://github.com/bluore/submit-plugin)
::github{repo="Bluore/submit-plugin"}
:::

使用时在控制台中输入命令：

```cmd
submit <filename>
```

:::warning
仅支持`AtCoder Beginner Contest`上的题目，且文件名使用`Competitive Programming Helper (cph)`插件的短文件名格式
:::