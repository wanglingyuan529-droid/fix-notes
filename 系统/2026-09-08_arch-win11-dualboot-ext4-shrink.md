# Arch + Windows 11 双系统安装记录（ext4 缩容两次踩坑）

> 日期：2026-09-08 · 环境：Arch Linux (x86_64) · 硬件：1TB SKHynix NVMe（GPT）

## 一、操作背景与目标

- 原有系统：单 Arch Linux（GPT 分区表，1GB EFI + 约 930GB Ext4 根分区）
- 目标：收缩 Ext4 根分区，腾出约 350GB 空闲空间安装 Windows 11，最终实现 GRUB 引导双系统
- 操作环境：Archiso Live U 盘、微 PE U 盘

## 二、Ext4 分区缩容全过程（含两次踩坑复盘）

### 核心原理

先收缩**文件系统**（resize2fs），再收缩**物理分区**（parted），顺序不能反；物理分区大小必须大于文件系统大小，否则触发超级块损坏报错。

### 第一次踩坑：单位进制完全混淆

**错误操作**

- resize2fs 用 `652G`（此处 `G` 为 GiB，1024 进制）
- parted 里设置分区结束为 `652GB`（十进制 GB，1000 进制），数值直接对等

**报错现象**

```
e2fsck: The physical size of the device is smaller than the superblock
Either the superblock or the partition table is likely to be corrupt!
```

**错误原因**

`resize2fs` 的 `G` 代表 **GiB（1024³ 字节）**，`parted` 的 `GB` 代表 **十进制 GB（1000³ 字节）**，直接数值对等会让物理分区比文件系统小几十 GB，立即触发损坏警告。

**回滚方案**

1. 进入 parted，执行 `resizepart 2` 直接回车，恢复分区到全盘大小
2. 执行不带参数的 `resize2fs /dev/nvme0n1p2`，自动扩展文件系统匹配分区大小
3. 执行 `e2fsck -f /dev/nvme0n1p2` 完整性校验，全部 Pass 即系统恢复完好

### 第二次踩坑：parted 扇区对齐损耗

**错误操作**

- 换算后设置 parted 结束为 `645GB`（理论上 600GiB ≈ 644.2GB，仅留 0.8GB 余量）
- parted 按 512B 扇区/1MiB 边界向下对齐舍入，实际分配的分区比输入值小几百 MB

**报错现象**

再次触发 superblock 大小不匹配警告，物理块数仅比文件系统少约 336MB。

**回滚方案**

同第一次：先恢复分区到全盘，再扩展文件系统并校验。

### 最终成功方案：预留充足对齐余量

1. **收缩文件系统到 600GiB**

```bash
resize2fs /dev/nvme0n1p2 600G
```

2. **验证文件系统块数（必须，上次就漏在这一步）**

```bash
tune2fs -l /dev/nvme0n1p2 | grep "Block count"
# 正确输出：157286400 blocks（4K 块）
```

3. **收缩物理分区（留足余量）**

```bash
parted /dev/nvme0n1
(parted) print
(parted) resizepart 2
End? 650GB   # 比理论值多留 5GB 余量，抵消扇区对齐损耗
(parted) print
(parted) quit
```

4. **最终完整性校验**

```bash
e2fsck -f /dev/nvme0n1p2
```

✅ 成功标准：5 个 Pass 全部通过，无 corrupt / superblock 报错。

**最终缩容后分区状态**

| 分区 | 大小 | 格式 | 作用 |
|------|------|------|------|
| /dev/nvme0n1p1 | 1GB | FAT32 | EFI 系统分区（双系统共用） |
| /dev/nvme0n1p2 | 约 649GB | Ext4 | Arch Linux 根分区 |
| 未分配空间 | 约 348GB | - | 用于安装 Windows |

## 三、Windows 11 微 PE 安装步骤

### 前置说明

微 PE 为手动安装，自由度高但容错率低，**严禁使用一键装机工具**，必须手动指定分区。

### 1. 分区识别与创建

打开 DiskGenius：

- 649GB 的 Ext4 分区会被误标为「损坏」，这是 Windows 对 Ext4 的兼容问题，**绝对不要执行修复/格式化**——Linux 下已通过 e2fsck 验证完好
- 选中磁盘末尾 348GB 未分配空间 → 右键新建分区 → 文件系统选 NTFS → 保存更改并格式化

### 2. WinNTSetup 安装配置

打开 WinNTSetup，三项核心配置：

1. **安装映像**：选择 U 盘中 `sources/install.wim`
2. **引导驱动器**：选择 0.99GB 的 FAT32（ESP）分区
   - ⚠️ **绝对禁止勾选「格式化引导分区」**，否则 Arch 引导直接报废
3. **安装驱动器**：选择刚才新建的 348GB NTFS 分区

确认无误后点击「安装」，等待文件释放完成后重启，拔掉 PE U 盘，系统会自动完成后续安装。

## 四、GRUB 双系统引导修复

Windows 安装后会强制覆盖 UEFI 引导优先级，开机默认进 Windows，属正常现象，Arch 系统数据完好。

### 修复步骤

1. 插入 Archiso U 盘，UEFI 启动进入 Live 环境

2. **挂载分区并进入 chroot**

```bash
mount /dev/nvme0n1p2 /mnt
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
```

3. **重装 GRUB 引导**

```bash
# 注意：架构是 x86_64，不要写成 x84_64
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

4. **生成启动菜单（自动识别双系统）**

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

✅ 成功标识：输出包含 `Found Windows Boot Manager`

5. **收尾重启**

```bash
exit
umount -R /mnt
reboot
```

重启后拔掉 U 盘，即可看到 GRUB 启动菜单，自由切换 Arch Linux 和 Windows。

## 五、核心避坑总结

1. **单位进制坑**：`resize2fs` 用 GiB（1024 进制），`parted` 用 GB（1000 进制），数值不能直接对等。
2. **扇区对齐坑**：parted 会向下对齐扇区，输入值必须比理论值多留 1~5GB 余量，宁大勿小。
3. **操作顺序坑**：永远先缩文件系统，再缩物理分区，顺序不能反，且每步之间要验证（tune2fs 查块数）。
4. **Ext4 识别坑**：Windows 下 DiskGenius 对 Ext4 误报损坏是常态，以 Linux 下 e2fsck 结果为准，不要修复/格式化。
5. **引导分区坑**：任何时候都不要格式化 ESP 分区，双系统共用该分区。
6. **命令拼写坑**：CPU 架构是 `x86_64`，不要写成 `x84_64`。

## 补充经验

- 缩容全程唯一不可逆风险是断电/中断移动数据块阶段，动手前务必先备份（rsync 到外部盘）
- 分区缩错、未挂载未重启时随时可救：parted 恢复全盘 → resize2fs 无参跟随 → e2fsck，三步回滚已验证有效
- 事前准备：`pacman -S os-prober` 并在 `/etc/default/grub` 中 `GRUB_DISABLE_OS_PROBER=false`，装完 Windows 后 `grub-mkconfig` 才会自动识别 Windows 条目
- Live 环境内存不足装不了图形版 gparted 时，命令行 parted + resize2fs 完全够用——前提是单位换算正确 + 每步验证