# 卸载 waybar 误伤共享依赖：壁纸快捷键、下拉终端、剪贴板历史集体失效

> 日期：2026-10-13 · 环境：Arch Linux (x86_64) · niri 26.04 + Noctalia shell · 背景：清理 Hyprland 遗留组件

## 现象

清理 Hyprland 时代的遗留工具（`waybar`、`rofi`、`mako`、`swaync`、`wofi`、`wlogout`、`cliphist`）后，
先后发现多个快捷键失效，且**报错各不相同、看不出是同一原因**：

**① `Mod+W` 壁纸选择器 —— 直接抛 Python 异常**

```
File "/home/iris/.config/noctalia/tools/wallpaper-picker.py", line 13, in <module>
    gi.require_version("GtkLayerShell", "0.1")
ValueError: Namespace GtkLayerShell not available
```

**② `Mod+grave` 下拉终端 —— 脚本内报命令找不到**

```
niri-scratch-toggle.sh: line 22: jq: command not found
niri-scratch-toggle.sh: line 24: jq: command not found
```

**③ `Mod+V` 剪贴板历史 —— 返回 ok 但窗口空白**

Noctalia 文案提示（事后才注意到）：
```
The cliphist watcher is not running; enable the service and copy content again
cliphist or wl-copy is missing; clipboard history is unavailable
```

**④ 媒体控制 —— playerctl 相关动作不可用**

## 根因

**`waybar` 是 Hyprland 时代的遗留组件，但它的依赖被 Nyxuri/Noctalia 复用。**
用 `pacman -Rns waybar` 移除时，Nyxuri 仍需要的依赖被当作"无人依赖的依赖"一起级联删除。

pacman 日志（权威证据）：

```
[2026-10-11T01:19:03+0800] [ALPM] installed gtk-layer-shell (0.10.1-1)
[2026-10-11T01:19:03+0800] [ALPM] installed playerctl (2.4.1-5)
[2026-10-13T00:21:43+0800] [ALPM] removed   gtk-layer-shell (0.10.1-1)
[2026-10-13T00:21:43+0800] [ALPM] removed   playerctl (2.4.1-5)
```

被连带移除、且 Nyxuri 实际需要的包：

| 包 | 谁在用 | 后果 |
|---|---|---|
| `gtk-layer-shell` | `wallpaper-picker.py`（`gi.require_version("GtkLayerShell","0.1")`） | 壁纸选择器崩溃 |
| `jq` | `niri-scratch-toggle.sh`（解析 `niri msg -j windows`） | 下拉终端失效 |
| `cliphist` | Noctalia `ClipboardService.qml` | 剪贴板历史空 |
| `playerctl` | Nyxuri 媒体控制动作 | 媒体键失效 |

> 关键点：这 4 个包在 `pacman -Qi waybar` 里**看不到**（它们是 waybar 的**间接**依赖），
> 只检查 waybar 自身无法发现。

## 试错（记录两个错误做法）

### 错误 1：只看被卸载包本身，没看它的依赖闭包

移除前检查了 "waybar 有没有被别的包依赖"（`Required By: None` → 判定安全），
但**没有反向检查 "waybar 的依赖有没有被别的包使用"**。
`gtk-layer-shell` 当时确实是 waybar 独占，所以 `-Rns` 判定它可删 ——
pacman 的判定没错，是**我的意图没告诉 pacman**。

### 错误 2：用 `comm` 对比包列表，导致误判"未被移除"

用"卸载前快照 − 当前包列表"求差集来定位被移除的包：

```bash
comm -23 <(sort packages-before.txt) <(pacman -Qq | sort) > removed.txt
```

结果 `grep '^gtk-layer-shell$' removed.txt` 返回 **0**，
一度让我误判"这些包不是被卸载的，是本来就没装" —— **方向完全错了**。

改用 **pacman 日志**提取（权威来源）后立刻定位：

```bash
grep '\[ALPM\] removed' /var/log/pacman.log | tail -200
```

日志里 `removed` 记录是卸载的**唯一可信来源**，比事后快照对比可靠。

## 修复

```bash
# 1. 装回被误删的 4 个依赖
sudo pacman -S gtk-layer-shell jq cliphist playerctl

# 2. cliphist 还需要监听服务（包内自带 systemd user unit，默认 disabled）
systemctl --user enable --now cliphist.service
```

验证：

```bash
ls /usr/lib/girepository-1.0/ | grep Layer   # → GtkLayerShell-0.1.typelib
jq --version                                 # → jq-1.8.2
bash ~/.config/niri/scripts/niri-scratch-toggle.sh kitty   # → 不再报 command not found
echo test | wl-copy && cliphist list         # → 有历史条目
```

## 验证

| 快捷键 | 修复后 |
|---|---|
| `Mod+W` 壁纸选择器 | ✅ 正常弹出 |
| `Mod+grave` 下拉终端 | ✅ 无报错 |
| `Mod+V` 剪贴板历史 | ✅ `cliphist list` 返回历史条目 |
| 媒体控制 | ✅ playerctl 可用 |

## 教训

1. **卸载前要查"依赖闭包"，不只是"谁依赖我"。**
   两个方向都要查：
   ```bash
   pacman -Qi <pkg> | grep '^Required By'      # 谁依赖我（卸了会不会破坏别人）
   pacman -Rs --print --print-nosave <pkg>     # 我会带走谁（级联列表先看一遍）
   ```
   **`-Rns` 之前务必先用 `--print` 看级联清单**，逐项确认里面没有别处在用的包。

2. **共享依赖要主动"摘出来"。**
   如果确认某个依赖需要保留，卸载前先把它标成显式安装，pacman 就不会连带删除：
   ```bash
   sudo pacman -D --asexplicit <pkg>     # 只改安装原因，不安装/不升级
   ```
   本次 `gtk-layer-shell` 等 4 个包如果提前这样标记，就不会出问题。

3. **定位"哪些包被删了"要用 pacman 日志，不要用快照求差集。**
   `comm` / 集合运算容易因排序、格式、Unicode 等问题静默失准；
   `/var/log/pacman.log` 的 `[ALPM] removed` 是权威记录。

4. **"遗留组件"的依赖可能被新环境复用。**
   换桌面环境（Hyprland → niri）时，旧组件（waybar）可以删，
   但**它的依赖不一定属于它** —— 删之前先反查每个依赖的反向依赖。

5. **报错形态不同 ≠ 原因不同。**
   Python 异常、shell 命令找不到、UI 空白、动作无响应看似四个问题，
   实际是同一个操作（一次卸载）造成的。**先查最近做过什么，再逐个查依赖。**
