---
title: Linux清除n天前的日志
date: 2020-04-09 20:00:00
tags:
    - Linux命令
categories:
    - 基础
    - Linux命令
---

```shell
find logs/ -type f -mtime +n -exec rm -f {} \;
```

 

