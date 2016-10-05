---
layout:     post
title:      "ubuntu下git以及github的简单使用"
subtitle:   "How build git and github?"
date:       2016-10-5 12:00:00
author:     "Bydas"
header-img: "img/home-bg-o.jpg"
tags:
    - github
---
# github基本功能

1、Repository:放项目的仓库

2、Fork：授权拷贝

3、Branch：Fork之后，在个人的github上出现同名Respository,是源Respository的分支（Branch）

4、Pull Request:推送请求，将修改后的代码推送给原作者

## git初始设置
* git config --global user.name "login.name"

* git config --global user.email "login.email"
## 初始化Git仓库（init）

* 执行命令 cd ～ 
  * 创建一个工作目录，比如 My_github
* cd ~/My_github
  * 如果是在windows下移植github到ubuntu，则把文件直接放在该目录下
* git init 
  * 显示 Initialized empty Git repository in $PROJECT/ .git/
* 上述结果表示在当前目录下创建一个.git的隐藏目录，这个就是所谓的git仓库，此时的～/My_github文件夹，就是工作树。

## 提交（commit）

接下来在本地
（待续。 。 。）
