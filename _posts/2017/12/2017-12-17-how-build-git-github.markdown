---
layout:     post
title:      "github 下搭载 jekyll 博客"
subtitle:   "How build git and github?"
date:       2017-12-17 22:00:00
author:     "Bydas"
header-img: "img/home-bg-o.jpg"
catalog: true
tags:
    - tools
    - github
---

## github 基本功能

* Repository : 放项目的仓库
* Fork : 授权拷贝
* Branch : Fork 之后，在个人的 github 上出现同名 Respository ，是源 Respository 的分支（Branch）
* Pull Request : 推送请求，将修改后的代码推送给原作者

#### 搭载 github + jekyll 博客

- New repository : such as name.github.io
- 将博客模板存放在该仓库之中

## git 配置

* git config --global user.name "login.name"
* git config --global user.email "login@email"

#### 始化Git仓库（init）

* 执行命令 cd ～ 
  * 创建一个工作目录，比如 My_github
* cd ~/My_github
  * 如果是在windows下移植github到ubuntu，则把文件直接放在该目录下
* git init 
  * 显示 Initialized empty Git repository in $PROJECT/ .git/
* 上述结果表示在当前目录下创建一个.git的隐藏目录，这个就是所谓的git仓库，此时的～/My_github文件夹，就是工作树。

#### 提交（commit）

接下来在本地仓库里添加一些文件，比如README或者提交一篇博文。进行如下操作

    $ git add README
    $ git commit -m "first commit" //提交到本地仓库，并没有提交到github上

#### 连接 github 并上传repository

首先在本地创建 ssh key

    $ ssh-keygen -t rsa -C "login@email"

成功执行会在~/下生成.ssh文件夹，打开进去id_rsa.pub,复制里面的key

回到github，进入Account Settings，左边选择SSH Keys,Add SSH Keys，title自命名，粘贴key,验证，在git bash下输入：

    $ ssh -T git@github.com

如果是第一次会提示是否continue，输入 yes就会看到：

You've successfully authenticated,but Github does not provide shell access.表示github连接成功

设置远程连接 github:

    $ git remote add origin git@github.com:user_name/Mytest.git
* git pull : 当更换电脑可以重新按上述步骤建立一个 git 库，将博客的代码库拉到本地进行编辑
* git push : 将修改后的文件推送到服务器，进行更新

将本地仓库的文件上传到github上：

    $ git push origin master
* git push 出现 error : The requested URL returned error: 403

  进入目录：.git/config

  ```
  [remote "origin"]  
      url = https://github.com/your_name/your_name.github.io.git  
  为
  [remote "origin"]
  url = https://your_name@github.com/your_name/your_name.github.io.git
  ```

## 总结

修改完代码后，使用：

* git status : 可以查看文件是否改动
* git diff filename : 查看文件修改了哪些地方
* git add : 添加要commit的文件，也可以用 git add -i 来智能添加文件
* git commit : 提交本次修改，-m 参数提交说明，可区分你对同个 git 库的多次 commit
* git push : 上传到 github，更新内容