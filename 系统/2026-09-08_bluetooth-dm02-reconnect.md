# 蓝牙耳机 DM02 突然连不上（删除配对后需重新发现）

> 日期：2026-09-08 · 环境：Arch Linux · 蓝牙芯片：MediaTek MT7921（USB）· 耳机：DM02 头戴式（双设备连接）

## 现象

DM02 蓝牙耳机一直能连电脑，某天突然连不上：连接建立后 1-2 秒自动断开，或直接报
`br-connection-page-timeout` / `Device not available`。手机能连 DM02，另一个耳机
OHR509 电脑也能连——只有「DM02 ↔ 电脑」这对组合不行。

## 排查链（按顺序）

1. **日志定位**：`journalctl -u bluetooth` 看到
   `Hands-Free Voice gateway failed connect to ...: Connection refused (111)`——
   耳机拒绝电脑的 HFP（免提通话）请求，怀疑是双设备耳机通话通道被手机占用。
2. **删除配对重配**：`bluetoothctl remove 0D:76:A3:6F:4D:86` 后重新扫描，却一直
   「扫不到」——一度误判为电脑蓝牙扫描故障（MTK 芯片的常见嫌疑），折腾了
   服务重启、rfkill、重载 btusb、USB unbind/rebind、清空 /var/lib/bluetooth 缓存、
   装 LTS 内核等一系列手段，全部无效。
3. **关键认知纠偏**：实际是**用户自己终端里 `bluetoothctl scan on` 扫描是正常的**
   （扫到了 OHR509 和舍友设备）——扫描从未坏，「扫不到」是环境/时机假象。
   真相：**DM02 被 remove 后，必须走「重新扫描发现 → 配对」这条完整路径**，
   而排查中一直卡在发现/配对的某个环节（配对模式灯语有时限、耳机被手机占用、
   报错误导等）。

## 根因

**蓝牙设备一旦被 `bluetoothctl remove`，电脑不再有它的记录，之后必须重新扫描
发现并配对**。期间任何一步没走通（耳机没在广播、配对模式超时、手机占用、误读
报错），都会表现为「以前能连、现在连不上」。DM02 本身和电脑蓝牙都是好的。

## 修复

```bash
# 让耳机空闲：手机蓝牙关闭 + 耳机完全关机再开机（普通开机，不必进配对模式）
bluetoothctl scan on          # 在用户终端执行，等 [NEW] Device ... DM02 出现
bluetoothctl pair 0D:76:A3:6F:4D:86   # 报 Already Exists 是正常的（记录已重建）
bluetoothctl trust 0D:76:A3:6F:4D:86
bluetoothctl connect 0D:76:A3:6F:4D:86
```

## 验证

- `bluetoothctl info` 三项全绿：Paired ✓ Trusted ✓ Connected ✓
- 播放音频正常出声（A2DP 建立），连接稳定不再自动断开
- Trusted 已设，以后开机自动连接

## 教训

1. **「蓝牙设备突然连不上」第一反应**：查它是不是被 remove 过 / 不在已知设备列表。
   被删设备必须重新扫描发现+配对，这条路径任何一环断掉都会连不上。
2. **报错文案会误导**：`Already Exists` 不是失败——检查 `bluetoothctl info` 的实际
   状态（Paired/Connected），可能连接已经建立。
3. **双设备耳机**的「双连」通常只指音乐通道（A2DP），通话通道（HFP）同时只服务
   一台设备且优先手机——手机占用时电脑连不上是耳机侧正常行为。
4. **配对模式灯语有时限**（一般 30 秒~2 分钟），超时自动退出；不同耳机「配对模式」
   的按键/灯语不同，以说明书为准。
5. **环境差异陷阱**：自动执行环境的 `bluetoothctl scan on` 结果可能与用户桌面终端
   不同（扫描会话/DBus 环境差异），别仅凭一条命令的输出断定「扫描坏了」——用
   差分实验（换设备对照）验证。
6. 诊断顺序参考：日志（journalctl -u bluetooth）→ 对照实验（手机/另一耳机）→
   状态检查（bluetoothctl info）→ 最小修复，别一上来就动内核/驱动。