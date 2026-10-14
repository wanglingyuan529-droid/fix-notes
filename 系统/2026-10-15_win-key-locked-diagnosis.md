# 所有 Mod 快捷键集体失效：真凶是键盘的 Win 键被硬件锁定，不是 niri

> 日期：2026-10-15 · 环境：Arch Linux · niri + Nyxuri dotfiles · ROG FALCHION ACE HFX
> 背景：Mod+Enter（开终端）、Mod+V（剪贴板）等所有 Mod 组合键突然全部无反应

## 现象

niri 里所有以 `Mod`（Win 键）打头的快捷键**同时失效**：`Mod+Enter` 不开终端、
`Mod+V` 不弹剪贴板。但鼠标点击、Chrome 里打字、应用内输入**全部正常**。

同时 `kitty` 进程确实活着，`niri msg` 也一切正常 —— 看起来像"合成器坏了"。

## 根因

**键盘的 Win 键在硬件层就没有发出任何信号**（ROG 的 Win 键锁定 / 游戏模式）。

决定性证据 —— 用 `evtest` 直接从内核读原始事件流，让用户按 `Win+V`：

```
Event: time ..., type 1 (EV_KEY), code 47 (KEY_V), value 1     ← V 收到了
Event: time ..., type 1 (EV_KEY), code 47 (KEY_V), value 0
（共 12 个 KEY_V 事件）
```

**`code 125`(KEY_LEFTMETA) / `code 126`(KEY_RIGHTMETA) 出现 0 次。**

也就是说：用户按的是 `Win+V`，系统只收到 `V`。Win 键的按键事件
**从未离开键盘**。同理按 `Win+Enter` 只收到 `KEY_ENTER`。

ROG 键盘有个"Windows 键锁定"功能（防止游戏中误触弹开始菜单），
开启后 Win 键被**固件层屏蔽**。因为是硬件行为，操作系统里
无论怎么查配置都查不出问题。

解锁：**`Fn` + `Win`**（按一次切换，再按一次恢复），键盘 LED 灯带会指示状态。

## 试错

这次绕了不少弯路，值得记录 —— **前三个方向的检查全部"正常"，恰恰说明问题在更底层**：

| 检查 | 结果 | 结论 |
|---|---|---|
| `niri validate` | ✅ 配置有效 | niri 配置没问题 |
| `niri msg action spawn -- kitty` | ✅ 成功开出窗口 | niri 的 IPC / 窗口管理没问题 |
| `grep 'Mod+Return' binds.kdl` | ✅ 绑定存在 | 快捷键定义没问题 |
| `xkbcli compile-keymap` 查 `LWIN` | ✅ `key <LWIN> { [ Super_L ] }` | XKB 映射没问题 |
| `pacman -Qq \| grep keyd/interception/xremap` | ✅ 一个都没装 | 无软件层重映射 |
| 检查 niri 打开的 input 设备 | ✅ 13 个设备全开着 | 权限/设备层没问题 |
| **`evtest` 抓原始事件流** | ❌ **Win 键零事件** | ✅ **真凶在这里** |

**最开始的误判**：怀疑是 niri 收不到键盘。但"其他按键能用、只有 Win 键不能用"
这个特征本身就排除了"键盘整体失灵"，应该更早把方向转向**单个按键**。

## 修复

键盘上按 **`Fn` + `Win`** 解锁（按一次切换状态）。

## 验证

解锁后重新抓一次事件流，确认 `code 125` 出现：

```bash
sudo evtest /dev/input/event2 | grep -E 'code 12[56]|KEY_LEFTMETA'
# 按一下 Win 键，应能看到：
# Event: time ..., type 1 (EV_KEY), code 125 (KEY_LEFTMETA), value 1
```

或者直接在合成器层面观察：`Win+Enter` 能开出终端、`Win+Space` 能切输入法。

## 教训

1. **"所有 Mod+X 失效"不等于"合成器坏了"。** 先分清是
   **修饰键没发出来**，还是**合成器没处理**。前者症状是"整个 Mod 家族一起死"，
   后者通常伴随其他异常。本例正是前者。

2. **`evtest` 是判断"按键到底有没有到系统"的唯一可靠手段。**
   配置、XKB、合成器日志全都可能显示"一切正常"，因为问题在它们上游：

   ```bash
   # 先找到键盘对应的 event 设备
   grep -l "" /sys/class/input/event*/device/name | while read f; do
     echo "$(basename $(dirname $f)): $(cat $f)"
   done | grep -i keyboard

   # 抓原始事件流（需要 root；不会独占设备，不影响正常使用）
   sudo evtest /dev/input/event2
   ```

3. **检查顺序应该从底往上，而不是从配置往下。**
   正确顺序：**硬件事件（evtest）→ XKB 映射 → 合成器配置**。
   我这次反着查，在前面几个"正常"上浪费了最多时间。

4. **游戏键盘/鼠标的固件功能会被误认成系统故障。**
   Win 锁定、宏模式、板载配置切换（`Fn`+组合键）都属于**操作系统完全看不见**的层面。
   外设是游戏定位时，先想想**有没有按到它的功能键**：
   尤其 `Fn` 组合、侧边拨钮、指示灯状态。

5. **排除法里被跳过的那一层，往往就是答案。**
   本例所有"配置层"检查都通过，唯一没查的是**最底层的原始输入**。
   当上层全部正常而现象依旧存在时，**必须往下钻一层**，而不是重复检查同一层。

## 附：本次用到的事件码

| 键 | code |
|---|---|
| `KEY_LEFTMETA`（左 Win） | 125 |
| `KEY_RIGHTMETA`（右 Win） | 126 |
| `KEY_ENTER` | 28 |
| `KEY_V` | 47 |
