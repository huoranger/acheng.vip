---
title: "Clash 不可用"
date: "2022-08-03T10:00:00+08:00"
slug: "clash-can-not-used"
tags: ["错误日志"]
---

今天来到公司，打开 Clash 发现不可用：

![](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220803102850161.png "clash错误截图")

去 `ping 127.0.0.1` 发现不通，于是去百度，解决方案是重新安装 `TCP/IP` 协议。命令如下：

```shell
netsh int ip reset c:\resetlog.txt
```

重新启动即可😎