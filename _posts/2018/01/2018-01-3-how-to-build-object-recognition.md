---
layout:     post
title:      "搭建基于树莓派和Tensorflow的无人机物体识别"
subtitle:   "How build object recognition?"
date:       2018-1-3 12:00:00
author:     "Bydas"
header-img: "img/about-bg-o.jpg"
header-mask: 0.3
catalog: true
tags:
    - AI    
    - Tensorflow
---

# 基于树莓派和Tensorflow的无人机物体识别

- 主要实现有以下三部分：      
  1. *在树莓派上通过图片采集学习实现物体识别*     
  2. *中文语音输出*
  3. *采集无人机上摄像头图像，通过无线传输，在树莓派上进行解码识别*

## Todo

```
1 看图说话功能
2 中文语音输出
3 图像无线传输
```

## 一、图像识别

 - 题目来源
     ​        人工智能（AI）已经进入一个新的蓬勃发展期。推动这一轮AI狂澜的是三大引擎，即深度学习（DL）、大数据和大规模并行计算，其中又以 DL 为核心。

     ​        机器学习是当前科技的热门话题，物体识别作为机器学习领域的热点，该项目主要研究在除了人脸识别之外，扩展到在更大的图库中识别一个指定的物体，从而实现人工智能的要求。

- 设计要求

    1. 小型无人机技术：PCB 设计以及飞控程序的编写

    2. 树莓派（Raspberry Pi）**维基百科**：基于Linux的单板机电脑，它只有巴掌大小，却有惊人的计算能力，你可以把它当做一台普通电脑。

       ​       *项目中使用树莓派最新的版本3,较前一代树莓派2，树莓派3的处理器升级为了64位的博通BCM2837，并首次加入了Wi-Fi无线网络及蓝牙功能。*

    3. Tensorflow: 一个由"Google大脑"团队的研究人员开发的机器学习库，Google遵循Apache License 2.0将其开源。该系统可以被用于语音识别、图片识别等多个领域。

       ​       *项目中我们主要用到一个叫做inception的模型（基于ImageNet数据集）。它可以完成物体识别，我们直接使用预训练好的模型。训练模型是个费时费力的工作，把智能当黑盒使用，就像电气时代的来临，变革社会的除了那些发电的人，那些懂得使用电力去改造传统行业，创造新的行业的人，也许对社会的变革更为深刻。尽管他们可能连卡诺循环都不知道，甚至不知如何将水蒸汽中的动能转换为功，进而驱动电机发电，从而造福大众。*

    4. ImageNet 数据集：包含约120万张训练图像、5万张验证图像和10万张测试图像，分为1000个不同的类别，用于机器学习中训练图像识别系统。

    5. 中文语音输出：在树莓派上安装 sdcv, 进行英汉翻译。用 ekho 进行中文转语音。Ekho（余音）是一个免费、开源的中文语音合成软件。它目前支持粤语、普通话（国语）、诏安客语、藏语、雅言（中国古代通用语）和韩语（试验中），英语则通过 eSpeak 或 Festival 间接实现。Ekho 支持 Linux、Windows 和Android 平台。

    6. 实验及结果分析：设计实验的测试环境，并且利用测试环境对系统的性能进行测试和总结，用 Tensorflow 对图像进行学习训练。

### 1.1 硬件准备

- 树 莓派3B
- 无人机+遥控板
- 蓝牙音箱
- 摄像头

### 1.2 软件准备

 - 操作系统 [raspbian](https://downloads.raspberrypi.org/raspbian/images/raspbian-2016-05-31/)
- Tensorflow  [轮子](http://tensordata.cn/tensorflow-0.9.0-cp27-none-linux_armv7l.whl)

### 1.3 树莓派无线配置

#### 1.3.1 登陆ssh

- **Window**下用cmd控制台输入 arp -a 命令得到局域网下与本机相连的 ip 地址
- **路由器** 下设置页面可通过 *DHCP服务器* 下的 *客户端列表* 找到名为 raspberrypi 的设备
- 登陆 **默认用户名** pi **密码** raspberry 

#### 1.3.2 图形界面vnc

 - VNC **维基百科**：VNC 借由网络，可发送键盘与鼠标的动作及即时的屏幕画面，VNC 与操作系统无关，因此可跨平台使用，由客户端，服务端和一个协议组成。

- **Window ** 下自行安装，我装的是 RealVNC

- **树莓派 ** 上安装 VNC

  ```
  $ sudo apt-get install tightvncserver # 安装
  $ vncserver :1   # 启用 ，之后在 vnc client里输入 ip:1即可进入图形界面
  $ vncpasswd      # 修改密码
  ```

### 1.4 Download 换源

 - [镜像主页](https://lug.ustc.edu.cn/wiki/mirrors/help/raspbian)
     *几番尝试，还是觉得国内源的中科大靠谱*

     ```
     #sudo vim /etc/apt/sources.list ,使内容变为
     deb http://mirrors.ustc.edu.cn/raspbian/raspbian/ wheezy main non-free contrib
     deb-src http://mirrors.ustc.edu.cn/raspbian/raspbian/ wheezy main non-free contrib
     ```

*执行 apt-get update 命令更新软件列表*

## 二、Tensorflow 安装及测试

### 2.1 安装

#### 2.1.1 下载好 **轮子** 开始安装

```
sudo pip install tensorflow-0.9.0-cp27-none-linux_armv7l.whl
```

#### 2.1.2 下载 **模型** *Inception-V3*

```
1 mkdir ~/tensorflow-related/model
2 cd ~/tensorflow-related/model
3 wget http://download.tensorflow.org/models/image/imagenet/inception-2015-12-05.tgz
4 tar xf inception-2015-12-05.tgz
```

*解压出来的文件*

> classify_image_graph_def.pb
>
> cropped_panda.jpg
>
> imagenet_2012_challenge_label_map_proto.pbtxt；
>
> imagenet_synset_to_human_label_map.txt
>
> LICENSE

- 加载模型

  ```
  mkdir  ~/tf
  cd /usr/local/lib/python2.7/dist-packages/tensorflow/models/image/imagenet
  python classify_image.py --model_dir ~/tf/imagenet #--model_dir 指定模型数据存放的目录
  ```


- 测试是否正常

  ```
  python /usr/local/lib/python2.7/dist-packages/tensorflow/models/image/imagenet/classify_image.py  --model_dir ~/tf/imagenet
  ```


- 如果所示如下则表示一切正常

>giant panda, panda, panda bear, coon bear, Ailuropoda melanoleuca (score = 0.89233)
>
>indri, indris, Indri indri, Indri brevicaudatus (score = 0.00859)
>
>lesser panda, red panda, panda, bear cat, cat bear, Ailurus fulgens (score = 0.00264)
>
>custard apple (score = 0.00141)
>
>earthstar (score = 0.00107)

### 2.2 测试