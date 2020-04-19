---
title: GIT分支规范
date: 2020-04-19 21:52:00
tags:
    - 规范
    - Git
categories:
    - 基础
    - 规范
---
## 1分支类型

### 1.1 主分支 master

> 概念：代码库应该有一个、且仅有一个主分支。所有提供给用户使用的正式版本，都在这个主分支上发布。

### 1.2 开发分支 develop

> 概念：主分支只用来分布重大版本，日常开发应该在另一条分支上完成。我们把开发用的分支，叫做Develop。

### 1.3 临时性分支

> 概念：除了常设分支以外，还有一些临时性分支，用于应对一些特定目的的版本开发。临时性分支主要有三种：

- 功能（feature）分支,它是为了开发某种特定功能，从develop分支上面分出来的。开发完成后，要再并入develop。

- 预发布（release）分支，它是指发布正式版本之前（即合并到Master分支之前），我们可能需要有一个预发布的版本进行测试。
     预发布分支是从Develop分支上面分出来的，预发布结束以后，必须合并进Develop和Master分支。
- 修补bug（hotfix）分支，软件正式发布以后，难免会出现bug。这时就需要创建一个分支，进行bug修补。
     它是从Master分支上面分出来的。修补结束以后，再合并进Master和Develop分支

## 2  分支命名规范

- feature分支命名规范
> feature/#+简要说明，简要说明尽量用1到3个单词描述本次修改的功能。如：feature/#send_coupon

- release分支命名规范
> release/版本号，需要修改主版本号和次版本号，修订版本号为0。如：release/2.1.0

- hotfix分支命名规范
> hotfix/版本号，需要修改修订版本号，主版本号和次版本号不变。如：hotfix/2.1.1

## 3 分支使用规范

- 新建仓库时只有一个master分支（用于发布），在master的基础上创建develop分支（开发主分支，主要用于合并其他分支）。

```
    git checkout -b develop master
    git push origin develop

```

- 每开发一个新功能时`必须`从develop上创建一个feature分支。

```
    git checkout -b feature/#send_coupon develop
    git push origin feature/#send_coupon

```

- 新功能开发完成后`必须`将feature分支合并回develop分支，并在此develop上`创建`release分支（用于测试和修改bug）。

```
    git checkout develop
    git merge --no-ff feature/#send_coupon
    
    git checkout -b release/2.1.0 develop
    git push origin release/2.1.0

```

- 完成测试并即将上线的时候`必须`将release分支合并到develop分支和master分支。

```
    git checkout develop
    git merge --no-ff release/2.1.0
    
    git checkout master
    git merge --no-ff release/2.1.0

```

- 遇到线上bug需要修复的时候`必须`从master上创建hotfix分支，测试完成后需将hotfix分支合并到develop分支和master分支。

```
    git checkout -b hotfix/2.1.1 master
    git push origin hotfix/2.1.1
    
    git checkout develop
    git merge --no-ff hotfix/2.1.1
    
    git checkout master
    git merge --no-ff hotfix/2.1.1

```

- **禁止**在master分支上进行开发。

- **禁止**将master分支和develop分支向其他分支合并。

- 在进行合并分支之前，`必须`对即将合并过去的分支（develop或master）进行拉取

```
    git pull origin develop
    
    git pull origin master
```
- 分支合并之后需要将`合并修改提交到远程`

```
    git push origin
```

- 临时性分支在确定没有用处的时候需要删除分支。

```
    git branch -d feature/coupon-171212

```

- git flow 图解：

![](http://base422.oss-cn-beijing.aliyuncs.com/git_branch.png)



## 4 分支使用最佳实践（参考）

- [git在团队中的最佳实践](https://www.cnblogs.com/cnblogsfans/p/5075073.html)
- [git分支管理策略](http://www.ruanyifeng.com/blog/2012/07/git.html)