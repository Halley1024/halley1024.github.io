---
layout: post
title: cargo学习笔记
---

# Cargo学习笔记

## 001 · Cargo介绍

什么是Cargo？Cargo是Rust的包管理器，这个包管理能够自动帮你解决项目使用的第三方包的导入与依赖。类似于Python中的conda和pip这类包管理。Rust引入这个有什么好处呢？我个人认为有下面几点，仅供参考：

- **自动化处理包依赖避免手工导入出错**：在复杂的项目中，如果我们使用`rustc`对rust项目进行编译时，手动显示输入依赖包，那么是十分困难。举个例子：

  ```rust
  // 打印 Hellow, World! ../src/main.rs
  fn main(){
      print!("Hellow, World!");
  }
  // 我们使用rustc src/main.rs 对其进行编译，将其转为二进制可执行文件
  // 此时就会在项目根目录..下出现 main.
  
  // 如果在引进
  ```

  

