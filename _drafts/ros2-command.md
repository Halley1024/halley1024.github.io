---
layout: post
title: ros2-command
---

# ROS2控制命令







创建新的一只海龟

```bash
ros2 service call /spwan turtlesim/srv/Spawn "{x: 2,y: 2, theta: 0.2, name: "abc"}"
```





记录和回放

```bash

ros2 bag record --topics  <topic> # you need record a topic. you can write the command such as, ros2 topic info, to look through the topics.
# 如果是Ubuntu26.04需要加上 --topics参数指定。


# when you want replay the trajectory, you can use command on shell bellow.
ros2 bag play
```



