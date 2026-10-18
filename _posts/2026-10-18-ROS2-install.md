---
layout: post
title: Rust智能指针详解
date: 2026-10-18 23:49 +0800
categories: [Blogs,ROS2]
tags: [Blogs]     # TAG names should always be lowercase
math: true
mermaid: true
---

# ROS2安装教程

ROS2支持Ubuntu和为Windows两个平台的安装。这里我只展示在Ubuntu平台上的安装教程。

在Ubuntu26.04上，我们需要安装ROS2长期维护的`lyrical luth`版本。安装详流程可以额参考官网[ROS2官网安装地址](https://docs.ros.org/en/lyrical/Get-Started/Installation/Ubuntu-Install-Debs.html)

## 001 · 系统设置

1. 首先确保自己的系统语言是`UTF-8`

   ```bash
   locale  # check for UTF-8
   sudo apt update && sudo apt install locales
   sudo locale-gen en_US en_US.UTF-8
   sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
   export LANG=en_US.UTF-8
   locale  # verify settings
   ```

2. 配置你的仓库

   因为ROS2支持Ubuntu26.04上的用户通过`apt`来安装ROS2 SDK，所以我们需要配置一下自己的 repository 。

   ```bash
   sudo apt install software-properties-common	# 
   sudo add-apt-repository universe	# 添加社区开的reposiotry
   ```

   接下来就是安装`ros-apt-source` 包。将下面一段输入到bash中去。

   ```bash
   sudo apt update && sudo apt install curl -y
   export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F'"' '{print $4}')
   curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
   sudo dpkg -i /tmp/ros2-apt-source.deb
   ```

3. 其他工作包

   ```bash
   sudo apt update && sudo apt install ros-dev-tools
   ```

------

## 002 · 安装ROS

这一步我们会通过正式安装ROS2的SDK。

```bash
# 1、第一部加载更新apt仓库内容
sudo apt update
# 2、对仓库里面的源apt包进行升级
sudo apt upgrade
# 3、安装可视化的ROS2软件包
sudo apt install ros-lyrical-desktop
# 或者安装基础包
sudo apt install ros-lyrical-ros-base
```

------

## 003 · 更新配置环境

```bash
# 直接在终端运行下面代码
source /opt/ros/lyrical/setup.bash
```

------

## 004 · 尝试连接

这一步需要打开两个终端，测试一下ROS2安装是否成功。

1. 先打开第一个`taker shell`，

   ```bash
   source /opt/ros/lyrical/setup.bash
   ros2 run demo_nodes_cpp talker	# 创建一个发送信息的taker
   ```

2. 再打开第二个`listener shell`

   ```bash
   source /opt/ros/lyrical/setup.bash
   ros2 run demo_nodes_py listener	# 创建一个接受消息的listener
   ```

执行上面步骤之后，会出现taker终端会一直发送消息，而listener终端会一直接受taker发送的消息。此时表示`ros2`安装成功。

其实到这一步，我们发现每次启动终端，Linux系统没有识别到`ros2`命令。因此我们需要将其加入到linux系统环境变量中，让shell终端能够自动识别。

我们可以输入一下命令：

```bash
# 打开终端
echo "sourece /op/ros/lyrical/setup.bash" >> ~/.bashrc	# 将显示的内容输入到~/.bashrc用户环境变量的文件中去
source ~/.bashrc
```

通过上面的操作，我们就能再每次打开shell终端的时候，就能调用`ros2`命令了。

------

## 005 · 卸载ROS2

假如你不需要ROS2了，那么可以执行下面这几条语句

```bash
# 自动移除apt中的包
sudo apt remove '~nros-lyrical-*' && sudo apt autoremove
# 卸载ros2
# sudo apt remove ros2-apt-source
sudo apt update
sudo apt autoremove
sudo apt upgrade # Consider upgrading for packages previously shadowed.
```

------

## 006 · 运行小海龟

除了上述的`talker`和`listener`可以测试`ros2`的安装是否成功。ROS2还提供了一个运行一个控制小海龟的仿真demo。

我们先打开终端1，输入下面命令

```bash
ros2 run turtlesim turtlesim_node
```

然后再打开另外一个终端，输入下面命令

```bash
ros2 run turtlesim turtle_teleop_key
```

此时就会出现

![image-20261018234024746](/home/halley/Documents/程序员技术/ROSA开发/turtle.png)

这样的一只小海龟，我们可以在控制终端，按下"上下左右"按键来控制这样的一只小海龟，这样就能控制其前进和后退以及旋转按钮。
