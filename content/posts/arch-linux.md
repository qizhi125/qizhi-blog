+++
date = '2026-01-11T12:00:00+08:00'
draft = false
title = 'Linux 运维学习：手动构建 Arch Linux 生产环境'
tags = ["Linux", "Arch", "DevOps", "Kernel"]
summary = "记录纯命令行环境下手动安装 Arch Linux 的核心流程，梳理磁盘分区、挂载逻辑与系统引导配置环节。"

+++

>手动安装 Arch Linux 是理解 Linux 底层组件（Kernel、Userland、Bootloader）协同工作流程的有效方式。
> 本文记录在命令行环境下的核心构建步骤及注意事项。

## 1. 连通性测试与分区准备

确保救援/安装环境网络连通，这是拉取系统核心包的前提。

```bash
ping -c 3 baidu.com
```

> **提示**：若提示 `Temporary failure in name resolution`，通常为 DNS 解析失败或网卡未启动，在 ISO 环境下可尝试执行 `systemctl start NetworkManager`。

确认目标磁盘设备（如 `/dev/vda`），使用 `cfdisk` 执行 GPT 分区。

**分区方案建议：**

- **vda1 (512M)**: EFI System。用于存放 UEFI 引导程序（GRUB）。
- **vda2 (2G)**: Linux swap。交换分区，防止物理内存耗尽触发 OOM 导致系统崩溃。
- **vda3 (剩余)**: Linux filesystem。根分区。

> **提示**：在 `cfdisk` 界面中，完成分区划分后必须选择 `Write` 并输入 `yes`，否则分区表不会真正写入磁盘。

## 2. 格式化与挂载顺序

根据 UEFI 规范及 Linux 运行要求，为不同分区写入对应的文件系统。

```Bash
mkfs.fat -F32 /dev/vda1              # 引导分区必须为 FAT32 格式
mkswap /dev/vda2 && swapon /dev/vda2 # 初始化并启用 Swap
mkfs.ext4 /dev/vda3                  # 根分区使用 ext4 格式
```

**挂载逻辑**：必须遵循先挂载根目录，再创建挂载点挂载子目录的顺序。

```Bash
mount /dev/vda3 /mnt          # 先挂载根目录
mkdir /mnt/boot               # 创建引导区挂载点
mount /dev/vda1 /mnt/boot     # 挂载引导分区
```

## 3. 部署核心系统与生成 fstab

在同步软件包之前，修改 `/etc/pacman.d/mirrorlist`，将国内镜像源（如清华源、阿里云源）置顶，并执行 `pacman -Syy` 刷新。

通过 `pacstrap` 将基础系统包安装至目标挂载点：

```Bash
pacstrap -K /mnt base linux linux-firmware vim networkmanager sudo man-db
```

> **提示**：`linux-firmware` 和 `networkmanager` 为必选项，缺失会导致新系统重启后无网卡硬件驱动或无法管理网络。

生成 `fstab` 文件以记录文件系统挂载信息：

```Bash
genfstab -U /mnt >> /mnt/etc/fstab
```

> **提示**：建议使用 `-U` 参数以 UUID 方式记录挂载点，避免未来硬件接口变动导致盘符（如 vda 变为 sda）漂移，引发系统无法启动。

## 4. 切换环境与系统配置

使用 `arch-chroot /mnt` 切换至新系统环境，此时的终端操作将直接作用于新系统。

**基础配置命令清单：**

```Bash
# 1. 时区与硬件时间同步
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc

# 2. 本地化配置 (取消 en_US.UTF-8 注释并生成)
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# 3. 主机名与 Root 密码
echo "arch-server" > /etc/hostname
passwd
```

## 5. 引导程序配置与收尾

安装并配置 GRUB 引导程序，这是系统能够独立启动的关键组件。

```Bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

> **提示**：执行 `grub-mkconfig` 时，务必观察终端输出。若包含 `Found linux image: /boot/vmlinuz-linux`，则证明引导程序已成功扫描并识别到刚才安装的内核。

配置完成后，执行安全的退出与重启流程：

1. `exit`（退出 chroot 虚拟环境）
2. `umount -R /mnt`（递归卸载挂载点，确保缓存数据完全刷入磁盘）
3. `reboot`（重启系统）

