---
title: "NCC 客端开发"
date: "2022-08-04T10:58:00+08:00"
slug: "ncc-devlopement"
tags: ["错误日志"]
---

问题1：8088 端口被占用用

![image-20220804110412460](../../../../static/images/NCC 客端开发/image-20220804110412460.png)

解决方法：杀掉 8088 端口占用程序

```sh
# 查看端口信息，获取pid --> 53860
netstat -aon|findstr "8088"

# 根据pid杀死程序
taskkill -pid 53860 -f
```

问题2：Oracle 数据库数据导入，在命令行下进行。

> 注意看注释！注意看注释！注意看注释！

