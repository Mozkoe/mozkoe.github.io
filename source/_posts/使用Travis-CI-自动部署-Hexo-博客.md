---
title: 使用 Travis CI 自动部署 Hexo 博客
categories: 未分类
date: 2016-06-17 18:43:21
tags: Hexo
description:
---

## Hexo 环境搭建
（依赖 node.js 环境）

### 全局安装 hexo
``` bash
npm install hexo-cli -g
```

### 初始化目录
``` bash
hexo init
```

## 手动更新之殇
- Hexo 静态博客每次更新完，都需要手动生成静态文件，再手动部署到 Github 上
``` bash
Hexo g
Hexo d
```
- 项目目录在执行 `Hexo init` 后，会首先清空整个文件夹，导致 git 文件被移除，无法进行版本控制

## 奇怪的想法

- 希望能在同一个 repositories 的 master 分支存放生成的静态文件，在另一个分支存放 Hexo 项目文件
- 需要能在项目文件 push 成功后，自动执行某些命令，并部署到 master 分支
（适用于 Github Pages）

感谢 [iissnan](http://notes.iissnan.com/2016/publishing-github-pages-with-travis-ci/) 同学提供的思路：[GitHub Webhooks](https://developer.github.com/webhooks/)

>这是 Github 提供的一种机制，使应用能与 Github 通讯。这种机制实际上就是 Pub/Sub，当 Github 监测到资源（如仓库）有变化就往预先设定的 URL 发送一个 POST 请求（Pub），告知变化情况，而后接收变化的服务器（Sub）即可做一些额外的事情。
>
> 这个思路需要有一个服务器并启动一个服务来接收 Github 的请求。这里又有种不同的策略，这两种策略都是基于源码放置在 Github 的前提。第一个是源码将最终文档直接部署在这台服务器上（如使用 Nginx），当接收到 Github 通知直接编译更新到服务器指定的文件夹下即可。另一种策略是当服务器接收到通知后编译更新，而后将编译后的版本提交到 Github 仓库的 gh-pages 分支，让 Github 做 Host。

在暂不考虑直接部署到服务器的前提下，选择使用 [Travis CI](https://travis-ci.org/) 来达成目标。

## 配置 Travis CI
直接将 Github 密码记录在 Travis CI 的配置文件 `.travis.yml` 里的显然还太年轻。
了解到 Github 支持生成 [Personal Access Token](https://github.com/blog/1509-personal-api-tokens) ，Travis CI 同时支持 [加密文件](https://docs.travis-ci.com/user/encrypting-files/) 与 [加密 Token Key](https://docs.travis-ci.com/user/encryption-keys/)后。剩下的只是顺水推舟了。


### 大致思路
- 获取 GitHub Personal Access Token
- 使用 Travis CI 的工具加密这个 Token，并保存到 .travis.yml 文件中
- 配置文件使用 Access Token

### 具体步骤

#### 获取 Github PA Token
前往 Github Setting 页，获取 Personal Access Token（该Token 仅出现一次，之后将无法再次显示）
需要注意的是，Personal Access Token 的权限应遵循最小原则。

#### 加密 Github PA Token
使用 Travis CI 的命令行工具加密 GitHub 的 Personal Access Token。这个工具是一个 gem 包，需要 Ruby 环境支持。

``` bash
# 安装 Travis CI 命令行工具
gem install travis

# 加密 Personal Access Token
travis encrypt -r Mozkoe/mozkoe.github.io GH_Token=XXX
```

第二条命令中 -r 后的参数是 GitHub 仓库的名字（<用户名>/<仓库名>）；GH_TOKEN 将作为环境变量使用。将这条命令输出的结果复制到 .travis.yml 文件下：

``` bash
env:
  global:
    - GH_REF: github.com/Mozkoe/mozkoe.github.io.git
    - secure: "XXX"
```

这个设置之中包含了项目仓库的地址（设置在 GH_REF 环境变量中）以及 Access Token （被加密了，设置在 GH_TOKEN 环境变量中）。
这两个环境变量将在 Travis CI Build 时被使用，用于往 GitHub gh-pages 分支推送。

#### 配置 .travis.yml
具体可以参考我的 [个人配置](https://github.com/Mozkoe/mozkoe.github.io/blob/gh-pages/.travis.yml)。

## git init 项目文件
配置好了 Travis CI，接下来我们用 git 重新管理本地项目文件。

进入项目根目录，执行

``` bash
git init
git remote add origin "remote-server"
```

这里的 "remote-server" 指代你的远程 git 仓库地址。

本地建立并切换到用于项目追踪的分支 `gh-pages`

### 遇到的坑
Hexo 主题我是从 Github 上 clone 过来的。这个主题，正好位于我的 Hexo 项目目录的某个子目录下。

- 在父目录执行 `git init` 后，子目录原有的 git 仓库将无法被收纳到父目录的版本控制程序当中。（一个目录，一个 git 原则）

- .gitignore 文件 "失效"，无法忽略某一个目录下的文件。

#### 错误的应对
首先我想到了直接手动删除子目录里 git 相关的文件与目录。 .git, .gitignore
发现这个目录仍然无法被 父目录（Hexo 项目目录）的 git 追踪。

自动构建已经完成，最后却卡在了 git 的使用上，心好累 o_O

#### 解决方案
- 感谢 [by liu](https://www.zhihu.com/question/24467417) 同学
>因为该文件夹已经被纳入了版本管理中，
>先清空该文件夹的本地缓存然后再添加就行了
>git rm -r --cached path

原来单纯的删除 git 相关的文件是不够的。

- 关于 .gitignore 文件 "失效" 的问题
> .gitignore 文件只对还没有加入版本管理的文件起作用，如果之前已经用 git 把这些文件纳入了版本库，就不起作用了
>解决:
>需要在 git 仓库库中删除该文件，并更新。
>然后再次 git status 查看状态，指定的被忽略文件不再显示状态。

好吧，原来是我之前就把项目目录的 public 目录提交到远端了，git 已经追踪了这个目录，而 .gitignore 无法对已加入版本管理的文件进行控制。

这下学到了。

#### 后续优化思路

Hexo 生成的静态文件放置 public 目录下，这个目录其实是不需要被追踪的。
所以我在项目根目录的 .gitignore 文件中添加了
``` bash
/public/*
```

暴露在 .travis.yml 文件中其他的私密信息（邮件地址，加密过的 key），能否也通过 Travis CI 的配置里的环境变量替代呢？

自动构建的过程每次都需要 20s - 50s 左右，能否通过优化 Travis CI 自动执行的命令行，增加缓存，来加快每次构建、部署的速度呢？









