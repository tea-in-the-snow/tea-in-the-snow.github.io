---
title: "Infer Labs"
description: Doing the labs in the infer framework to take the participant through the basics of using Infer's Abstract Interpretation (AI) framework for building compositional abstract interpreters. 
date: 2024-08-13T11:54:38+08:00
comments: true
draft: true
categroies:
  - Static Analysis
tags:
  - infer
---

## 在实验室的服务器上搭建`infer`

环境：Debian12 in docker on Ubuntu22.04.

运行`./build_infer.sh`编译infer，运行`make devsetup`设置开发环境。

## Warm up: running, testing, and debugging Infer

- 根据README文件中提供的步骤在html文件和命令行的输出中添加`Hi infer`。（将对应的函数添加到对应的位置）并且重新编译运行。
- 注意，`infer-out`文件夹会在运行infer二进制文件的当前目录下生成。
