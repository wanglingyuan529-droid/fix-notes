# fcitx5 5.1 主题不生效（配置文件格式变更）

日期：2026-09-10 · 环境：Arch Linux + Hyprland (Wayland) + fcitx5 5.1.21-1 + fcitx5-material-color 0.2.1-2

## 背景与目标

装了 fcitx5-material-color 皮肤（11 款 Material 主题），想把输入法候选框换成紫罗兰（Material-Color-DeepPurple）。改配置文件 + 重启 fcitx5 后**外观纹丝不动，一直是默认配色**。

## 最终成果（结论先行）

- **紫罗兰主题生效**，候选框正常渲染，重启后保持。
- 根因是 **fcitx5 5.1.x 的 classicui.conf 从「[Appearance] 段 + Theme 键」改成了「扁平键值」**——旧的 `[Appearance]` 写法整个被静默忽略，连系统主题都切不动。
- 最可靠的操作方式：**用 D-Bus 配置接口设置，让 fcitx5 自己写盘**，一步到位且格式永远正确。

## 排障历程（现象 → 根因 → 修复）

### 1. `fcitx5-remote -r` 并没有真正重启进程
- **现象**：改完配置执行 `fcitx5-remote -r`，`ps` 显示 fcitx5 进程启动时间没变。
- **根因**：`-r` 在该版本未生效（或只请求未执行），配置未被重新加载。
- **修复**：`pkill -x fcitx5` 后重新 `nohup fcitx5 -d &`。
- **踩坑**：`pkill -f 'fcitx5'` 会把**命令行里含 fcitx5 字样的当前 shell 一起杀掉**（命令自杀，输出为空）；要用 `-x` 精确匹配进程名。

### 2. 配置被读取了，但主题名没生效（本记录核心）
- **现象**：strace 显示 fcitx5 确实 `openat` 了 `~/.config/fcitx5/conf/classicui.conf`，但启动时只加载 `default` 主题；把 Theme 改成系统内置 `default-dark` 也无效。
- **排查**：D-Bus 查询运行时配置 `GetConfig("fcitx://config/addon/classicui")` → 主题枚举列表里 **11 款 Material 主题全部在列**（说明主题本身被识别），但 `Theme: 'default'`——磁盘配置没进运行时。
- **根因**：fcitx5 5.1.x 的 classicui.conf **不再使用 `[Appearance]` 段**。用 `SetConfig` 让 fcitx5 自己写盘后，得到正确格式是**顶层扁平键值**（带中文注释）：

```ini
# 主题
Theme=Material-Color-DeepPurple
# 深色主题
DarkTheme=default-dark
# 跟随系统浅色/深色设置
UseDarkTheme=False
# 当被主题和桌面支持时使用系统的重点色
UseAccentColor=True
```

- **修复（推荐）**：不手写文件，直接用 D-Bus 设置：

```bash
gdbus call --session --dest org.fcitx.Fcitx5 --object-path /controller \
  --method org.fcitx.Fcitx.Controller1.SetConfig \
  "fcitx://config/addon/classicui" \
  "<{'Theme': <'Material-Color-DeepPurple'>}>"
```

> SetConfig 第二参数是 variant，gdbus 里要**双重尖括号**嵌套，否则报 `can not parse as value of type 'v'`。

- **验证**：设置后 `GetConfig` 返回 `Theme: 'Material-Color-DeepPurple'`，且 fcitx5 自动把正确格式写回磁盘（下次启动也保持）。

### 3. 查运行时状态用 D-Bus，别猜
- `CurrentUI` → 返回 `classicui`（确认候选框由谁渲染）
- `GetConfig("fcitx://config/addon/classicui")` → 当前值与枚举列表
- `DebugInfo` → 各输入法上下文与前端
- 配置文件路径：`~/.config/fcitx5/conf/<组件名>.conf`（插件自己的配置）

### 4. 需要源码确认时用镜像拉
- 下载 fcitx5 5.1.21 源码确认配置结构：`https://gh-proxy.com/https://github.com/fcitx/fcitx5/archive/refs/tags/5.1.21.tar.gz`（7.7MB）
- 关键代码位置：`src/ui/classic/classicui.h` 的 `ClassicUIConfig`（Theme/DarkTheme/UseDarkTheme 三个键的默认值）；`src/ui/classic/classicui.cpp` 的 `reloadConfig() → readAsIni(config_, "conf/classicui.conf")`

## 教训清单（下次避免）

1. **fcitx5 5.1.x 的 classicui.conf 是扁平键值，没有 `[Appearance]` 段**——网上 5.0 时代的教程写法已失效。
2. **让程序自己写配置文件**：用 D-Bus SetConfig（或 GUI 改一次），再照它写盘的内容抄格式，永远正确。
3. **`fcitx5-remote -r` 不保证重启进程**；验证用 `ps` 看启动时间，硬重启用 `pkill -x fcitx5 && fcitx5 -d`。
4. `pkill -f` 会匹配自身命令行，自杀坑；精确进程名用 `pkill -x`。
5. 主题枚举正常（GetConfig 里能看到全部主题）≠ 配置生效，两者分开查。

## 回滚

```bash
# 恢复默认主题：用 D-Bus 设回 default，或直接删配置
rm ~/.config/fcitx5/conf/classicui.conf
pkill -x fcitx5; nohup fcitx5 -d &
# 卸载皮肤
sudo pacman -R fcitx5-material-color
```
