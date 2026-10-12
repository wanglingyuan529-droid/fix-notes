# 卸载 KDE Plasma 与 Hyprland：五个会把系统搞坏的坑

> 日期：2026-10-13 · 环境：Arch Linux · 目标：只保留 niri，清掉 KDE 与 Hyprland
> 结果：包数 985 → 836（移除 149 个），依赖数据库零错误，niri 全程未中断

## 现象 / 目标

系统里同时装了 KDE Plasma、Hyprland、niri 三套桌面。目标是卸载前两套，只留 niri。
看似只要 `pacman -Rns plasma-meta`，实际连踩五个坑，其中一个会**直接搞坏音频**。

## 坑 1：`xdg-desktop-portal-impl` 是虚拟依赖，卸载顺序错了会拖垮 niri

```
niri Depends On: xdg-desktop-portal-impl        ← 虚拟包
  ├── xdg-desktop-portal-hyprland (Provides: xdg-desktop-portal-impl)  ← Hyprland 装的
  └── xdg-desktop-portal-gtk      (Provides: xdg-desktop-portal-impl)
```

两个包都提供同一个虚拟依赖，niri 的 `Required By` 里会**同时**列出它们。
直接删 hyprland/kde 的 portal 会被 pacman 拒绝（"breaks dependency required by niri"）。

**先查 niri 实际用哪个后端**：

```bash
cat /usr/share/xdg-desktop-portal/niri-portals.conf
# [preferred]
# default=gnome;gtk;          ← niri 用的是 gtk/gnome，不是 hyprland！
```

**正解**：先显式标记 gtk 为提供者，再移除另一个：

```bash
sudo pacman -D --asexplicit xdg-desktop-portal-gtk
sudo pacman -R xdg-desktop-portal-hyprland
```

## 坑 2（最危险）：移除 KDE 会连带删除 `pipewire-pulse`，**系统直接没声音**

```
pipewire-pulse         Required By: plasma-pa（KDE 音量控制）
pipewire-audio         Required By: pipewire-pulse
pulse-native-provider  Required By: plasma-pa
```

`plasma-pa` 是 KDE 的音量控制小部件，跟着 plasma-meta 一起走 —— 而 `pipewire-pulse`
（PulseAudio 兼容层，浏览器/播放器靠它出声）**只被它依赖**，于是被一起删掉。

**后果**：`pacman -Rns plasma-meta` 之后，所有走 PulseAudio 接口的程序**静默无声**。

**正解**：卸载前把这几个包标成显式安装，pacman 就不会连带删除：

```bash
sudo pacman -D --asexplicit pipewire-pulse pipewire-audio pulse-native-provider
```

**卸载列表从 129 个降到 109 个** —— 这 4 个包的差别就是"有声/无声"。

## 坑 3：`-Rns` 级联会带走重量级工具

模拟 `pacman -Rns xdg-desktop-portal-kde` 显示会删除 **150 个包**，其中包含：

```
qt6-multimedia   ripgrep     upower     protobuf
xorg-xrdb        accountsservice        libxpm
```

原因：它是 niri 的（虚拟）依赖，`-Rns` 顺着依赖链把一批"看起来没人要"的包全清了。

**正解**：改用**不带 `-s` 的 `-R`**，成组移除，让 pacman 不做递归清理：

```bash
sudo pacman -R pkg1 pkg2 pkg3 ...     # 不用 -Rns
```

## 坑 4：其他被 KDE 独占、但本身有用的工具

`pacman -Rs --print plasma-meta` 列出的 129 个包里混着这些：

| 包 | 被谁依赖 | 为什么不能删 |
|---|---|---|
| `ddcutil` | powerdevil | niri 的亮度脚本用它控制外接屏 |
| `noto-fonts-emoji` | plasma-desktop | emoji 渲染 |
| `libnotify` | signon-ui | `notify-send` 通知 |
| `networkmanager-qt` | plasma-nm | NetworkManager 的 Qt 绑定 |
| `clinfo` | kinfocenter | OpenCL/GPU 诊断 |

同样用 `pacman -D --asexplicit <pkg>` 逐个保下来。

## 坑 5：移除父包会让子包的"安装原因"降级，之后可能被误删

移除 `breeze` 后，正在使用的 GTK 光标主题 `breeze-cursors`
安装原因从 `Explicitly installed` **降级为依赖**，出现在 `pacman -Qdt`（孤立包）列表里 ——
下次有人跑 `pacman -Rns $(pacman -Qdtq)` 就会被误删。

**正解**：发现后重新标记：

```bash
sudo pacman -D --asexplicit breeze-cursors breeze-icons
```

## 修复（完整流程）

```bash
# 0. 先记录基线，便于回滚
pacman -Qq  > packages-before.txt
pacman -Qeq > packages-explicit-before.txt

# 1. 保护共享依赖（改安装原因，不安装任何东西）
sudo pacman -D --asexplicit \
    pipewire-pulse pipewire-audio pulse-native-provider wayland-utils \
    ddcutil noto-fonts-emoji libnotify networkmanager-qt clinfo

# 2. 让 gtk 接管虚拟 portal 依赖
sudo pacman -D --asexplicit xdg-desktop-portal-gtk

# 3. 模拟，务必逐项检查级联清单
pacman -Rns --print plasma-meta

# 4. 移除（分组、不用 -Rns 处理 KDE 框架部分）
sudo pacman -Rns --noconfirm plasma-meta
sudo pacman -R --noconfirm <残留 KDE 框架包...>

# 5. 收尾：重新加固被降级的包
sudo pacman -D --asexplicit breeze-cursors breeze-icons
```

## 验证

```bash
pacman -Dk                        # → No database errors
pgrep -x niri noctalia sddm       # → 全部存活
pgrep -x pipewire wireplumber     # → 音频守护在跑
wpctl status | grep -A2 Sinks     # → 有输出设备
curl -sI https://www.baidu.com    # → 网络正常
ls /usr/share/wayland-sessions/   # → 只剩 niri.desktop
pacman -Qq | grep -c '^hypr'      # → 0
pacman -Qq | grep -c '^plasma'    # → 0
```

## 教训

1. **`-Rns` 之前必须 `--print` 模拟，并逐项检查级联清单。**
   本次靠它发现 `pipewire-pulse` 会被删 —— 否则系统会静默失去声音，
   而且很难联想到是"卸 KDE"造成的。

2. **"谁依赖我"和"我会带走谁"是两个方向，都要查。**
   ```bash
   pacman -Qi <pkg> | grep '^Required By'    # 谁依赖我
   pacman -Rs --print <pkg>                  # 我会带走谁
   ```

3. **虚拟依赖要显式指定提供者**，用 `pacman -D --asexplicit <provider>`。

4. **不要用递归依赖闭包判断"哪些必须保留"。**
   我一开始写了脚本递归求闭包，结果把 `glibc`、`systemd`、`zlib` 等基础包全卷进来，
   产生 **331 个假阳性**，完全误导判断。
   正确做法：**只检查集合外的直接依赖方**（不递归）。

5. **移除父包后要复查 `pacman -Qdt`**，把仍在使用的包重新标为显式。

6. **卸载桌面环境不影响登录管理器。** 本例 `sddm` 的依赖链
   （`bash glibc qt6-base xorg-server...`）不含任何 KDE 包，
   所以卸 KDE 不影响登录。**但要先验证**，不要假设。
