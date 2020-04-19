---
title: GIT仓库命名规范
date: 2020-04-19 21:50:00
tags:
    - 规范
    - Git
categories:
    - 基础
    - 规范
---

- **【强制】**对于存放微服务代码的GIT仓库,采用以Avatar为前缀的命名规范,命名风格采用大驼峰式风格,例:AvatarCoupon,AvatarWallet
- **【强制】**对于存放中间件代码的GIT仓库,采用以middleware为后缀的命名规范,命名风格以 - 为分割符,如是管理后台中间件,前接backend,如是TO C端中间件,前接frontend,例:coupon-backend-middleware,coupon-frontend-middleware
- **【强制】**对应存放前端代码的Git仓库，采用fe-`${项目名}-${endpoint}`格式进行命名，项目名称多个单词之间使用分割线(-)连接，`endpoint`可选值为有`pc`（默认PC端应用）、`mob`（移动端应用）、`wechat`（小程序）