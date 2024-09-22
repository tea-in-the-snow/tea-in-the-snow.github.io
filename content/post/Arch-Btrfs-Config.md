---
title: "Arch 使用 Btrfs 文件系统的相关配置"
description: "在 Arch 上使用 Btrfs 文件系统，并使用 snapper 创建和管理快照的记录。"
date: 2024-09-22T10:17:26+08:00
hidden: false
comments: true
draft: false
categories: [Tools and Configuration]
tags: [Arch, Btrfs, snapper]
---

Btrfs 是一种现代的 COW（Copy on Write）文件系统，具有子卷和快照功能，对于 Arch 这样的滚动更新的系统来说能够提供很好的系统备份和还原的方案。

一下的记录仅仅涉及到 Arch 安装和配置过程中涉及到 Btrfs 的部分，其余安装步骤可以参考[官方文档](https://wiki.archlinuxcn.org/wiki/%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97)

## 创建分区和建立子卷

首先使用 cfdisk 工具分配将要安装 Arch Btrfs 文件系统的分区，这里我分配的分区为`/dev/nvme1n1p4`

### 格式化 Btrfs 分区

```terminal
mkfs.btrfs -f -L Arch /dev/nvme1n1p4
```

挂载刚刚创建的 Btrfs 分区：

```terminal
mount /dev/nvmen1p4 /mnt
```

### 建立子卷

Btrfs 可以为子卷建立快照，其中，子卷的一份快照仅包含这个子卷的内容，对于下面嵌套的子卷，快照只会包含嵌套子卷的挂载点，而不会包含嵌套子卷的内容。利用这个特性，可以对整个系统进行快照，而排除一些目录（如用户数据等）。

子卷的规划参考了 [openSUSE](https://zh.opensuse.org/SDB:BTRFS#.E9.BB.98.E8.AE.A4.E5.AD.90.E5.8D.B7)。

- `@` 对应 `/`
- `@home` 对应 `/home`，/home 目录被自动排除在快照之外以避免因回滚造成的数据丢失。
- `@root` 对应 `/root`，/root 目录因为和 home 分区相同的原因被排除在快照之外。
- `@tmp` 对应 `/tmp`，/tmp 目录包含了应用程序产生的临时文件和缓存文件，它被排除在快照之外。
- `@opt` 对应 `/opt`，第三方产品的安装目录经常是 /opt 。它被排除在快照之外以避免因为回滚而造成卸载应用。
- `@var` 对应 `/var`，这个文件夹包含了各式各样的文件，如日志文件、临时缓存、存放于 /var/opt 的第三方产品，并且是许多虚拟机镜像和数据库默认的存储位置。因此将它排除在快照之外，让许多数据避免在系统回滚的过程中被损坏。
- `@usr_local` 对应 `/usr/local`，在手动安装应用时会使用到该目录。它被排除在快照之外以避免因为回滚而造成卸载应用。

创建子卷：

```terminal
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@root
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@opt
btrfs su cr /mnt/@var
btrfs su cr /mnt/@usr_local
```

创建子卷结束后，卸载先前挂载的 /mnt 分区：

```terminal
umount /mnt
```

挂载子卷：

```terminal
mount -m /dev/nvme1n1p4 -o subvol=@,compress=zstd:1,noatime /mnt
mount -m /dev/nvme1n1p4 -o subvol=@home,compress=zstd:1,noatime /mnt/home
mount -m /dev/nvme1n1p4 -o subvol=@root,compress=zstd:1,noatime /mnt/root
mount -m /dev/nvme1n1p4 -o subvol=@tmp,compress=zstd:1,noatime /mnt/tmp
mount -m /dev/nvme1n1p4 -o subvol=@opt,compress=zstd:1,noatime /mnt/opt
mount -m /dev/nvme1n1p4 -o subvol=@var,compress=zstd:1,noatime /mnt/var
mount -m /dev/nvme1n1p4 -o subvol=@usr_local,compress=zstd:1,noatime /mnt/usr/local
```

- `compress=zstd:1` 指定了所使用的压缩算法以及压缩等级。
- 在默认情况下，每次访问文件时，文件系统都会更新文件的一些时间戳，这在绝大多数时间都是没有必要的，通过 `noatime` 可以禁止文件读取时时间元数据的更新，可以显著提高文件系统的性能。

为 `/var` 文件夹关闭 COW 功能：

```terminal
chattr +C /mnt/var/
```

至此，Btrfs 文件系统的初始化已经完成，后续正常进行系统的安装和配置。

### 注意事项

在生成 fstab 时系统会自动给子卷加上 `subvolid`，这会导致快照的恢复出现问题，建议删除。

生成 fstab 的命令：

```terminal
genfstab -U /mnt >> /mnt/etc/fstab
```

删除的命令：

```terminal
sed -i 's/subvolid=.*,//' /mnt/etc/fstab
```

## 使用 snapper 创建和管理快照

`snapper` 是 openSUSE 团队的 Arvin Schnell 所开发的工具，可以用于管理 Btrfs 子卷和快照。

通过 pacman 安装 snapper：

```terminal
sudo pacman -S snapper
```

创建 snapper 配置文件：

```terminal
sudo snapper -c root create-config /
```

编辑配置文件 `/etc/snapper/configs/root`：

```config
# start comparing pre- and post-snapshot in background after creating
# post-snapshot
BACKGROUND_COMPARISON="yes"

# run daily number cleanup
NUMBER_CLEANUP="yes"

# limit for number cleanup
NUMBER_MIN_AGE="3600"
NUMBER_LIMIT="40"
NUMBER_LIMIT_IMPORTANT="10"

# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="3600"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="5"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_QUARTERLY="0"
TIMELINE_LIMIT_YEARLY="5"

# cleanup empty pre-post-pairs
EMPTY_PRE_POST_CLEANUP="yes"

# limits for empty pre-post-pair cleanup
EMPTY_PRE_POST_MIN_AGE="3600"
```

启动相关 systemd 服务：

```terminal
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

以上服务分别配置 snapper 根据时间线每小时，每天和每年创建快照，以及自动清理快照。

也可以安装 `btrfs-assistant` 包，使用 GUI 管理 Btrfs 快照和子卷。

## pacman snapper 工具

安装软件包 `snap-pac`：

```terminal
sudo pacman -/s snap-pac
```

pacman hook，让 snapper 自动在 pacman 命令执行前后创建 pre 和 post 快照，就像 openSUSE 在 zypper 命令前后所做的那样。

## 在 grub menu 中显示快照

安装 `grub-btrfs` 包：

```terminal
sudo pacman -S grub-btrfs
```

grub-btrfsd 服务需要 inotify-tools，可能需要另外安装：

```terminal
sudo pacman -S inotify-tools
```

启动服务：

```terminal
sudo systemctl enable --now grub-btrfsd.service
```
