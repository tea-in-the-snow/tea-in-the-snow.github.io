---
title: "i3wm 安装与配置记录"
description: "在 Arch 上安装与配置 i3wm 的记录。"
date: 2024-09-21T23:22:14+08:00
hidden: false
comments: true
draft: true
categories: [Tools and Configurations]
tags: [i3wm, Arch]
---

在把 Gnome，KDE，Xfce 都体验过一遍后，由于想要尽可能少的离开键盘去摸鼠标，于是产生了使用平铺式窗口管理器的想法。原先选择的是 Hyprland，毕竟确实非常美观，但是 Xwayland 的缩放问题导致诸如 VSCode，Chrome 等应用缩放后会显得很糊，只能使用禁用 Xwayland 缩放的暂时解决方案。在加上 Hyprland 现在为止还没有发布稳定版，bug 还是挺多的，于是最终放弃了 Hyprland，打算选择不那么漂亮和炫酷，但是久经考验的 i3wm。

## 安装 i3wm

当下我的系统环境：

![pre installation system info](pre-system-info.png)

使用 pacman 直接安装：

```terminal
sudo pacman -S i3-wm
```

## 一些基本组件的安装和配置

### 应用程序启动器

使用 `rofi` 应用程序启动器：

```terminal
sudo pacman -S rofi
```

修改配置文件中对 mod+d 的映射：

```terminal
bindsym $mod+d exec --no-startup-id rofi -show drun
```

### 壁纸

使用 `feh` 设置壁纸：

```terminal
sudo pacman -S feh
```

在配置文件的最后添加：

```config
exec_always feh --bg-scale ~/Pictures/wallpaper/19.jpg
```

### 分辨率和缩放

安装 `xorg-xrandr` 软件包：

```terminal
sudo pacman -S xorg-xrandr
```

可以通过 `xrandr` 命令查看和设置显示器的分辨率。

对于缩放，在 XResources 中设置缩放：

```terminal
Xft.dpi: 144
```

i3 默认的 dpi 为 96，所以设置1.5倍缩放应该设置 dpi 为144。

### 输入法

我之前已经配置好了 Fcitx5 输入法（fcitx5-rime + 雾凇拼音），所以直接配置 Fcitx5 在 i3 启动时自动启动就行。

设置 `Fcitx5` 自动启动：

```config
exec --no-startup-id fcitx5 -d
```

## 键位映射和美化

```config
bindsym $mod+h focus left
bindsym $mod+j focus down
bindsym $mod+k focus up
bindsym $mod+l focus right
```

关闭 i3 标题栏：


