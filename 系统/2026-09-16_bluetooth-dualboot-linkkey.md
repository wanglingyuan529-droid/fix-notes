# 双系统蓝牙冲突：Windows 用过耳机后 Linux 连不上（需删设备重配）

> 日期：2026-09-16 · 环境：Arch Linux + Win11 双系统 · 蓝牙芯片：MediaTek MT7921（USB）· 耳机：DM02 头戴式（双设备连接）

## 现象

- Windows 接入过 DM02 后，切到 Linux 连不上；反之亦然。
- Linux 能扫描到设备，但**连接建立后 1~2 秒自动断开**。
- Windows 连接很快，Linux 侧扫描明显偏慢。
- 唯一的出路：把设备从当前系统**删除（remove）再重新配对**，之后就正常，直到下次切换系统。

## 根因：Link Key（链路密钥）双系统冲突

蓝牙 BR/EDR 配对成功后，电脑和耳机**各存一份 link key**（128 位认证密钥）。
双系统共用同一个物理适配器 = 同一个蓝牙地址（BD_ADDR），但配对信息各自独立存储：

| 系统 | 存储位置 |
|---|---|
| Linux (BlueZ) | `/var/lib/bluetooth/<适配器地址>/<设备地址>/info` 的 `[LinkKey]` 段 |
| Windows | 注册表 `HKLM\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys\<适配器地址>\` |

问题在于**耳机端只能为同一个地址记住一份 key**：

1. Windows 配对/连接 → 耳机端存的是 Windows 的 key。
2. 切到 Linux，Linux 拿着自己存的旧 key 去认证 → **认证失败** → 表现为「连上马上掉」。
3. remove + 重新配对 → 生成新 key，Linux 和耳机两端都写入新值 → 又正常了。
4. 再切回 Windows，Windows 拿旧 key 去连，耳机端已被 Linux 覆盖 → Windows 也连不上。**谁最后配对谁生效**。

> 这也解释了 2026-09-08 那次「DM02 连上 1-2 秒自动断开」——同一根因家族：
> 电脑端记录与耳机端 key 不一致时，唯一出路就是重新配对（在任一端）。

## 为什么 Windows 快、Linux 慢

- **Windows 连得快**：耳机端 key 若还是 Windows 的，直接认证通过，无需重配；且 Windows 蓝牙驱动是厂商私有、扫描策略激进、缓存设备列表。
- **Linux 扫描慢**：BlueZ 的 inquiry 扫描节奏保守 + MT7921U（mt76 驱动）已知扫描偏慢；另外 Arch Wiki 特别提到 **MT7921/MT7961 双系统下 Windows 与 Linux 固件版本不同会导致适配器状态异常**——先确保两边固件都是最新的（Arch 侧升级 `linux-firmware` 即可，Windows 侧更新官方驱动）。
- 若日志出现 `command tx timeout` 一类，可试内核参数 `btusb.enable_autosuspend=n`（电源管理导致的适配器异常）。

## 修复

### 方案 A（日常兜底，现状即可）

切换系统后连不上就 remove + 重配（流程同 2026-09-08 笔记）。能解决问题，但每次切换系统都要重来一遍。

### 方案 B（治本：让双系统共享同一份 link key）

一次配置，之后两边都能免重配直连。思路：让 Windows 和 Linux 使用**同一个 key**，耳机端存的也就是这个 key，两边认证都能通过。

#### 前置检查（4 项，缺一不可）

1. **MAC 稳定性**：比对 Linux（`bluetoothctl devices`）与 Windows 里记录的 DM02 地址是否一致。部分设备（如 Logitech MX Master）每次与新系统配对会递增 MAC 的一个字节——若 DM02 也这样，key 同步会失效，退回方案 A。
2. **确认 DM02 音频只用经典蓝牙 key**：看 Linux 侧 `/var/lib/bluetooth/<适配器>/<DM02-MAC>/info` 是否有 `[LinkKey]` 段（音频 A2DP/HFP 只依赖它）。若同时有 `[IdentityResolvingKey]` 等 LE 段，属 Bluetooth 5.1 设备——LE 部分不影响音频，可忽略（bt-dualboot 也不支持同步 LE）。
3. **Windows 分区不能被 BitLocker 加密**（chntpw 读不了加密 hive；加密则改走下面的手动路线 2）。
4. **先保证 Linux 侧当前能正常连接 DM02**（以 Linux 的 key 为基准向外同步）。

#### 路线 1：bt-dualboot（推荐，一条命令，Linux → Windows）

[bt-dualboot](https://github.com/x2es/bt-dualboot) 直接读 Linux 的 key 写进 Windows 注册表 hive，自动处理编码/字节序，无需手抄、无需多次重启。

```bash
# Arch：装依赖
sudo pacman -S python-pip chntpw
sudo pip install bt-dualboot

# 挂载 Windows 分区（可写）。先看工具识别到哪些：
sudo bt-dualboot --list-win-mounts
# 若没自动识别，手动挂载后指定：
sudo mount -o remount,rw /mnt/win/路径

# 干跑预览 → 正式同步
sudo bt-dualboot -l              # 列表里 DM02 应显示 "Needs sync"
sudo bt-dualboot --sync-all      # 或 --sync <DM02的MAC> 只同步耳机
```

- 会要求选择备份策略：`--backup`（默认，备份 Windows 注册表 hive 到 /var/backup/bt-dualboot）或 `--no-backup`。chntpw 是非官方工具，**建议保留备份**，Windows 确认能连后再删。
- 同步完**重启进 Windows**，直接连接 DM02，不应再要求重配。

#### 路线 2：手动（Windows → Linux，Arch Wiki 官方流程）

1. Windows 侧：从 [PsTools](https://download.sysinternals.com/files/PSTools.zip) 解出 `PsExec64.exe`，管理员 CMD 运行：
   ```
   .\PsExec64.exe -s -i regedit.exe
   ```
   （注册表键只能被 SYSTEM 账户访问，必须用 PsExec 提权）
2. 导航到 `HKLM\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys\<适配器MAC>`，找到名为 DM02 MAC 的 REG_BINARY 值（16 字节 link key），右键导出 .reg 拷到 Linux。
3. Linux 侧：编辑
   ```
   sudo nano /var/lib/bluetooth/<适配器MAC>/<DM02-MAC>/info
   ```
   把 `[LinkKey]` 段的 `Key=` 换成 .reg 里的值：**大写、去掉空格**（Arch Wiki 注：Windows 侧直接复制即可，无需反转字节序——反转是 macOS plist 才需要的操作）。
4. `sudo systemctl restart bluetooth`，重新连接；若图形界面连不上，先完整重启一次。

> 也有一条 Linux 端读取捷径：`chntpw -e /路径/Windows/System32/config/SYSTEM`，`cd CurrentControlSet\Services\BTHPORT\Parameters\Keys`，用 `hex <值名>` 直接看 key，省得来回拷贝。BitLocker 加密分区除外。

#### 方案 B 的维护成本（重要）

- 同步之后**不要**再在任意单边删设备重配——重配会生成新 key，两系统 key 再次分叉，需重跑一次同步。
- 新设备加入时重复一次流程（Linux 配对 → `--sync-all`）。
- 若 key 已一致仍「连上几秒掉」，多半是 BlueZ 5.83+ 的 multipoint bug（见方案 C）。

### 方案 C（若「连上几秒掉」在 key 已一致时仍出现）

BlueZ 5.83+ 有个已知 bug：**multipoint（双设备连接）耳机**会因一次多余的认证尝试失败而断开（[bluez#1330](https://github.com/bluez/bluez/issues/1330)），DM02 正是双设备耳机。临时解法：`sudo systemctl restart bluetooth`。

## 验证

```bash
# 连接瞬间抓认证失败证据（应看到 Authentication Failure / 相关报错）
journalctl -u bluetooth -f
# 或抓 HCI 事件（需 root）
sudo btmon

# 确认配对状态
bluetoothctl info <设备MAC>   # Paired ✓ Connected ✓
```

## 教训

1. 双系统蓝牙问题的第一嫌疑是 **link key 不一致**，不是硬件坏、不是扫描坏、不是驱动坏。
2. 「连接上 1~2 秒自动掉」几乎就是认证失败的指纹；配好 key 或重配后立刻消失。
3. 谁最后配对谁生效——这是双系统蓝牙的「内存」规则。
4. 想一劳永逸就同步 key（方案 B）；不想折腾就接受「切换系统后重配一次」的日常流程（方案 A）。
5. MT7921 双系统下先把两边蓝牙固件都更新到最新，能排除适配器层的干扰。

## 参考

- Arch Wiki: [Bluetooth#Dual boot pairing](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing)、[Device connects, then disconnects](https://wiki.archlinux.org/title/Bluetooth#Device_connects,_then_disconnects_after_a_few_moments)、[MT7921/MT7961 dual boot](https://wiki.archlinux.org/title/Bluetooth#Mediatek_MT7921_or_MT7961_on_dual_boot_with_windows)
- [bluetooth-dualboot](https://github.com/nbanks/bluetooth-dualboot) / [bt-dualboot](https://github.com/x2es/bt-dualboot)
- 关联笔记：[2026-09-08_bluetooth-dm02-reconnect.md](https://github.com/wanglingyuan529-droid/fix-notes/blob/main/%E7%B3%BB%E7%BB%9F/2026-09-08_bluetooth-dm02-reconnect.md)
