+++
date = '2026-01-12T12:57:56+08:00'
draft = true
title = 'Hello World'
+++
# 🐧 Linux 运维实战演练：Arch Linux 从零构建 (PDF 完整复刻版)

**文档来源**：实战演练 - Arch Linux 从零构建.pdf

**演练目标**：掌握 Linux 服务器初始化、磁盘管理、核心安装及引导配置。

**实战环境**：GNOME Boxes / Arch Linux ISO

---

## 🟢 Part 1：环境初始化 (Infrastructure)

### 第一节：网络连通性检查 (Network Check)

> **📄 场景模拟**：拿到一台新服务器 or 排查故障时，第一件事永远是检查网络。

#### 1. 执行检查命令

```bash
ping -c 3 baidu.com
```

- **参数含义**：`-c 3` 表示只发送 3 个包，防止无限刷屏。

#### 2. 结果分析 (Result Analysis)

![image-20260107235347395](/image/arch/image-20260107235347395.png)

- **✅ 正常现象**：
    - 显示 `64 bytes from ...`
    - **含义**：DNS 解析正常，网卡已获取 IP，外网连通。
- **❌ 异常现象**：
    - 显示 `Temporary failure in name resolution`
    - **含义**：DNS 解析失败，通常意味着网卡未启动或未配置 DNS。

#### 3. 故障排除 (Troubleshooting)

如果 Ping 不通，在 Arch ISO 环境下尝试以下方案：

- **方案 A**：启动网络管理器 `systemctl start NetworkManager`
- **方案 B**：手动使用工具 `iwd` (无线) 或 `dhcpcd` (有线)。

### 第二节：磁盘分区管理 (Disk Partitioning)

> **📄 场景模拟**：面对一块全新的硬盘（或虚拟机磁盘），规划存储空间。
>
> **🔧 核心工具**：`lsblk` (查看), `cfdisk` (图形化操作)。

#### Step 1: 检查磁盘状态 (Inspection)

在操作前，必须确认硬盘设备名，防止误格盘。

```bash
lsblk
# 预期：看到名为 vda 或 sda 的磁盘，且没有全部分配。
```

![image-20260107235558541](/image/arch/image-20260107235558541.png)

- **预期结果**：

    - 看到名为 `vda` (虚拟机磁盘) 或 `sda` (物理机 SATA 磁盘) 的设备。

    - TYPE 为 `disk`。

    - SIZE 应该显示总容量（如 40G），且下方没有任何分支（即没有 `part`），说明是空盘。

#### Step 2: 执行分区操作 (Execution)

使用图形化工具 `cfdisk` 进行交互式分区。

```bash
cfdisk /dev/vda
```

##### 1. 选择分区表 (Label Type)

- **操作**：选择 **`gpt`**。
- **🧠 面试考点**：*为什么选 GPT？*
    - **答**：GPT 是现代服务器标准，支持 >2TB 硬盘，必须配合 UEFI 启动模式使用。

##### **2. 详细分区方案 (Partition Scheme)**：

| **分区编号** | **容量大小** | **类型 (Type)**      | **作用 (面试必问)**                                       |
| ------------ | ------------ | -------------------- | --------------------------------------------------------- |
| **vda1**     | **500M**     | **EFI System**       | **引导区**。存放 Bootloader (GRUB)，UEFI 固件开机只认它。 |
| **vda2**     | **2G**       | **Linux swap**       | **交换分区**。内存不够时作为备用内存，防止系统 OOM 崩溃。 |
| **vda3**     | **剩余所有** | **Linux filesystem** | **根分区 (/)**。存放系统文件、日志和用户数据。            |

![image-20260107235943386](/image/arch/image-20260107235943386.png)

##### **3. 写入磁盘 (Persist)**：

- **操作**：选中底部的 `[ Write ]` -> 输入 `yes` -> 回车。
- **⚠️ 警示**：如果不执行此步，之前的操作仅停留在内存中，不会生效。

#### Step 3: 最终验证 (Verification)

再次执行 `lsblk` 确认分区结构。

```bash
lsblk
```

![image-20260108000159505](/image/arch/image-20260108000159505.png)

- **验收标准 (Expected Output)**：

  ```tex
  NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
  vda    254:0    0   40G  0 disk
  ├─vda1 254:1    0  500M  0 part
  ├─vda2 254:2    0    2G  0 part
  └─vda3 254:3    0 37.5G  0 part
  ```

### 第三节：格式化与挂载 (Formatting & Mounting)

> **📄 场景模拟**：将划分好的“毛坯房”装修（格式化文件系统）并分配门牌号（挂载目录）。

#### 1. 格式化文件系统 (Formatting)

根据不同分区的用途，刷入不同的文件系统。

```bash
# 1. 引导分区 -> 必须 FAT32 (UEFI 规范要求)
mkfs.fat -F32 /dev/vda1  [cite: 74, 75]

# 2. 交换分区 -> 制作并启用 Swap
mkswap /dev/vda2         [cite: 76]
swapon /dev/vda2         [cite: 78]

# 3. 根分区 -> 使用 EXT4 (最稳健的 Linux 文件系统)
mkfs.ext4 /dev/vda3      [cite: 80, 81]
```

![image-20260108001625918](/image/arch/image-20260108001625918.png)

#### 2. 挂载分区 (Mounting)

**⚠️ 运维铁律**：挂载必须遵循**父子顺序**，先挂根目录，再挂子目录。

```bash
# 第一步：挂载根目录 (必须最先执行！)
mount /dev/vda3 /mnt       [cite: 105]

# 第二步：创建挂载点 (有了根目录才能建文件夹)
mkdir /mnt/boot            [cite: 107]

# 第三步：挂载引导分区
mount /dev/vda1 /mnt/boot  [cite: 110]
```

![image-20260108002149581](/image/arch/image-20260108002149581.png)

------

## 🟡 Part 2：核心系统构建 (System Build)

### 第四节：镜像源优化 (Mirror Configuration)

> **📄 场景模拟**：服务器位于海外，或者默认源下载速度仅几 KB/s，导致业务部署超时或失败。
>
> **🔧 运维动作**：切换国内高速源（清华/阿里/中科大），并配置多级备用源以实现**容灾**。

#### 1. 编辑镜像列表

使用 vim 打开 pacman 的镜像配置文件。

```Bash
vim /etc/pacman.d/mirrorlist
```

#### 2. 配置策略 (Interview Point)

在文件最顶端添加国内源（优先级最高）：

```Bash
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.aliyun.com/archlinux/$repo/os/$arch
```

![image-20260108091549622](/image/arch/image-20260108091549622.png)

- **操作要点**：
    - 修改完后必须执行 `pacman -Syy` 强制刷新数据库。

#### 3. 故障排查经验 (Story)

- 问题描述：初始化环境时，发现默认源下载速度极慢（约 20KB/s）。
- **解决方案**：分析网络链路，通过编辑配置引入清华和阿里云镜像 。
- **结果**：下载速度提升至宽带满速，体现了运维人员对**服务可用性**和**网络优化**的意识 。

### 第五节：基础系统安装 (Base System Installation)

> **场景模拟**：根据业务需求，定制化安装操**📄 场景模拟**：根据业务需求，向空硬盘中定制化灌入操作系统核心组件 。作系统核心组件。

#### 1. 执行安装命令

```Bash
pacstrap -K /mnt base linux linux-firmware vim networkmanager sudo man-db
```

![image-20260108091716446](/image/arch/image-20260108091716446.png)

#### **2. 组件解析 (Component Analysis)** 面试时需解释“为什么装这些包”：

| **组件名称**               | **作用 (面试话术)**                                      |
| -------------------------- | -------------------------------------------------------- |
| **linux & linux-firmware** | **内核与驱动**。没有 firmware，网卡和显卡可能无法工作 。 |
| **vim**                    | **运维的手术刀**。系统装好后，必须有编辑器来修改配置 。  |
| **networkmanager**         | **网络管理工具**。提供 `nmcli` 命令，方便后续配置网络 。 |

------

### 第六节：系统交接与穿越 (System Handoff)

> **📄 场景模拟**：安装文件已就绪，现在需要生成“地图”（fstab）并正式“入驻”（chroot）新系统 。

#### **1. 生成挂载表 (Fstab Generation)**

**作用**：告诉 Linux 内核，开机时每个分区该挂载到哪里。如果不做这一步，重启后系统无法识别硬盘 。

```Bash
genfstab -U /mnt >> /mnt/etc/fstab
```

#### **2. 验证挂载表 (Verification) —— 必做！** **运维铁律**：修改配置后必须 Review。

```Bash
cat /mnt/etc/fstab
```

![image-20260108092539976](/image/arch/image-20260108092539976.png)

- **检查点**：
    - 看到 `UUID=... / ext4` (根分区)
    - 看到 `UUID=... /boot vfat` (引导分区)
    - *面试加分项*：解释为什么用 **UUID**（防止插拔硬盘导致盘符漂移）。

#### 3. 穿越进入新系统 (Chroot)

**作用**：Change Root。将当前的根目录从“安装光盘”切换到“硬盘里的新系统” 。

```Bash
arch-chroot /mnt
```

## 🟡 Part 3：系统核心配置 (Configuration)

### 第七节：系统身份确立 (System Identity)

> **场景模拟**：进入新系统后，必须确立它的“身份”（时间、语言、名字）。

#### **1. 基础配置命令清单**

- **时区与硬件时间**：

  ```bash
  ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  
  hwclock --systohc
  ```

- **本地化 (Locale)**：

  ```bash
  locale-gen # 生成 en_US.UTF-8 和 zh_CN.UTF-8  
  echo "LANG=en_US.UTF-8" > /etc/locale.conf
  ```

- **主机名与网络身份**：

  ```bash
  echo "QIZHI" > /etc/hostname
  ```

- **Root 密码**：

  ```bash
  passwd
  ```

![image-20260108093936807](/image/arch/image-20260108093936807.png)

### 第八节：引导程序安装 (The Bootloader - GRUB)

> **核心考点**：这是决定电脑开机能否找到 Linux 的关键一步。

#### 1. 安装 GRUB 到 EFI 分区

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

- **✅ 关键验收标准 (Key Output)**：
    - 终端显示：**`Installation finished. No error reported.`**
    - *面试话术*：看到这句话，说明 GRUB 的 `.efi` 文件已经成功写入了 EFI 分区，主板已经认识这个系统了。

#### 2. 生成 GRUB 配置文件

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

![image-20260108094332151](/image/arch/image-20260108094332151.png)

**✅ 关键验收标准 (Key Output)**：

- 终端显示：`Found linux image: /boot/vmlinuz-linux`
- 终端显示：`Found initrd image: /boot/initramfs-linux.img`
- *面试话术*：这步是为了让 GRUB 自动扫描 `/boot` 目录，找到 Linux 内核的位置，并生成启动菜单。如果没扫到内核，开机就会进 `grub rescue` 模式。

------

## 🏁 Part 4：收尾与重启 (Finalize)

> **当前操作指引**：你的系统已经全部装好了！现在我们需要优雅地退出并重启。

请在终端依次执行以下最后三个命令：

1. **退出新系统** (回到安装盘环境)：

   ```bash
   exit
   ```

2. **卸载分区** (确保数据写入磁盘，防止文件损坏)：

   ```bash
   umount -R /mnt
   ```

3. **卸载分区** (确保数据写入磁盘，防止文件损坏)：

   ```bash
   reboot
   ```


![image-20260108133246371](/image/arch/image-20260108133246371.png)