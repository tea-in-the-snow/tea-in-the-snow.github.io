---
title: "用 systemd 配置 Docker 使用代理访问 Docker Hub 拉取镜像"
description:
date: 2024-08-23T12:51:02+08:00
hidden: false
comments: true
draft: false
categories:
  - Tools and Configurations
tags:
  - Docker
---

由于 tuna 等国内开源镜像站突然全部停止了对 Docker Hub 的镜像服务，因此需要配置 Docker 使用代理访问 Docker Hub 拉取镜像。

简单的使用`export http_proxy`命令修改命令行环境的代理是不会影响 Docker 对 Docker Hub 的访问的，需要使用 systemd 来配置 Docker 的代理服务。

以下方案在 Fedora workstation 40 中成功应用。

在目录`/etc/systemd/system/docker.service.d`（如果没有，创建路径）中创建`proxy.conf`文件，内容为：

```conf
[Service]
Environment="HTTP_PROXY=http://xxx:xxxx"
Environment="HTTPS_PROXY=http://xxx:xxxx"
```

xxx 部分填写自己的代理地址。

配置好后重启服务：

```terminal
sudo systemctl daemon-reload
sudo systemctl restart docker
```

可以通过下面的命令查看是否配置成功：

```terminal
sudo systemctl show --property=Environment docker
```
