---
layout: post
title: 如何使用Rust去写ROS2节点
date: 2026-10-21 01:10 +0800
categories: [Blog,ROS2]
tags: [Blogs]     # TAG names should always be lowercase
math: true
mermaid: true
---

# 如何使用Rust去写ROS2节点

## 001 · 安装Rust依赖包

ROS2节点除了使用C++和Python来写，还可以使用rust语言来编写。同C++一样需要安装一些配置依赖。

这里假设我们已经成功安装好了ROS2环境以及python环境。那么我只需要继续安装rust依赖即可

```bash
pip install colcon-cargo colcon-ros-cargo
```

ros2编译是通过colcon来编译的，所以我们只需要安装colcon-cargo依赖，让colcon能够正常编译我们所写的rust代码。



## 002 · 创建工作空间

同c++和python一样，我们需要先创建一个工作空间，创建工作空间之后，在空间下创建src源代码目录

```bash
mkdir -p ~/ros2_rust_ws/src
cd ~/ros2_rust_ws/src
cargo new --bin --edition 2021 rust_demo	# 正常使用 cargo 创建rust项目
cd rust_demo
```

这里我们就创建好了rust功能包，整体项目录如下

```bash
rust_ros_ws/
└── src/
    └── rust_demo/
        ├── Cargo.toml
        ├── package.xml
        └── src/
            └── main.rs
```

创建好了之后，我们需要在`cargo.toml`文件中加入`rclrs`和`ros-env`rust包。

```rust
[package]
name = "rust_demo"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "talker"
path = "src/main.rs"

[dependencies]
rclrs = "0.8"
ros-env = "0.3"
```

不同于C++和python，使用`ros2 pkg --build-type ament_make <pakage-name>`和`ros2 pkg --build-type ament_make <pakage-name>`。

rust语言需要自己在`rsut_demo`下创建`package.xml`文件，用来表示所创建的功能包的相关信息。

```xml
<?xml version="1.0"?>
<package format="3">
  <name>rust_demo</name>
  <version>0.1.0</version>
  <description>Simple ROS 2 node written in Rust</description>

  <maintainer email="developer@example.com">Developer</maintainer>
  <license>Apache-2.0</license>

  <depend>rclrs</depend>
  <depend>rosidl_runtime_rs</depend>
  <depend>example_interfaces</depend>

  <export>
    <build_type>ament_cargo</build_type>
  </export>
</package>
```

完成上面步骤之后，那么就可以说我们正式创建了一个ros2 node项目。

## 003 · 编写rust代码

这里我们只做一个简单的示例，在main.rs文件中写下下面代码

```rust
use rclrs::*;	// 使用rclrs包
use ros_env::example_interfaces::msg::String as RosString;
use std::{thread, time::Duration};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 初始化 ROS 2
    let context = Context::default_from_env()?;
    let executor = context.create_basic_executor();

    // 创建节点：rust_talker
    let node = executor.create_node("rust_talker")?;

    // 在 /rust_chatter 话题上发布字符串
    let publisher =
        node.create_publisher::<RosString>("rust_chatter")?;

    let mut count = 0_u64;

    while context.ok() {
        let mut msg = RosString::default();	// 生成一个String类
        msg.data = format!("Hello from Rust! {count}");	// 在msg加入信息

        publisher.publish(&msg)?;	// 使用发布者节点对外进行发布
        println!("发送：{}", msg.data);

        count += 1;
        thread::sleep(Duration::from_secs(1));
    }

    Ok(())
}
```

在ros2中，topic这种发布/订阅模型是多对多方式。我们可以在功能包中创建一个订阅和一个发布。不仅如此，同一个功能包中可以创建多个node。但是这里我们只用一个node且只实现其发布功能



## 004 · 启动rust node

写好rust代码之后，我们就能启动由rust编写的node了

1. 首先我们需要对rust进行编译，这里我们使用colcon而不是rust原生的cargo来编译

   ```bash
   pwd	# ~/ros2_rust_demo 检查是不是在工作目录下，确保colcon的编译产物与src源码目录同级
   
   colcon build	# 自动生成build, install, log三个目录
   ```

2. 然后我们需要执行`install/setup.sh`，告诉ros2命令从哪里去找node文件

   ```bash
   source ~/rust_ros_ws/install/setup.bash
   ```

3. 启动node，操作还是与c++和python一样，使用`ros2 run`来启动

   ```bash
   ros2 run rust_demo talker
   # 终端会显示
   # 发送：Hello from Rust! 0
   # 发送：Hello from Rust! 1
   # ...
   
   # 通过 ros2 node info 就可查看是否启动了节点
   ros2 node info
   ```

至此，一个简单的rust开发node应用的基本流程就完成了。对比有编制的c++和python而言，rust开发node应用确实要准备更多前置步骤。