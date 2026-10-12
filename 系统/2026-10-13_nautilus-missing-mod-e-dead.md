# 快捷键绑定的程序从未安装：nautilus 缺失导致 Mod+E 无反应

> 日期：2026-10-13 · 环境：Arch Linux · niri + Nyxuri dotfiles

## 现象

按 `Mod+E` 没有任何反应（不报错、不开窗口、无提示）。

## 根因

Nyxuri 的 `binds.kdl` 把 `Mod+E` 绑定到 `nautilus`，但**该程序从未安装过**：

```
$ grep -n 'Mod+E' ~/.config/niri/binds.kdl
37:    Mod+E { spawn "nautilus"; }

$ command -v nautilus
(无输出)

$ pacman -Qi nautilus
error: package 'nautilus' was not found
```

niri 的 `spawn` 失败是**静默的** —— 找不到可执行文件不会弹错误，只是什么都不发生，
所以这个问题可以潜伏很久没人发现。

排查时先用 `pgrep`/`niri msg windows` 确认没有窗口产生，再直接手工执行绑定命令，
才看到真实原因。

## 修复

```bash
sudo pacman -S nautilus
```

验证：

```bash
command -v nautilus          # → /usr/bin/nautilus
ldd $(command -v nautilus) | grep -c 'not found'   # → 0（依赖完整）
nautilus &                   # 能启动；niri msg windows 出现 org.gnome.Nautilus
```

## 验证

`Mod+E` 正常打开文件管理器。

## 教训

1. **`spawn` 类绑定失败是静默的。** 快捷键没反应时，第一步是**把绑定的命令手工执行一遍**，
   而不是怀疑 niri 或键盘：
   ```bash
   grep -n 'Mod+E' ~/.config/niri/binds.kdl     # 先看绑的是什么
   <粘出来直接运行>                              # 再看能不能跑
   ```

2. **dotfiles 里的绑定不等于依赖已装。** 作者的配置假设了一组程序存在，
   但安装脚本未必会把所有可选程序都装上。换环境后应**逐个键位验证命令是否存在**：
   ```bash
   # 从 binds.kdl 提取所有 spawn 的命令并检查
   grep -oP 'spawn\s+"\K[^"]+' ~/.config/niri/binds.kdl | sort -u | while read c; do
     c=${c%% *}; command -v "$c" >/dev/null 2>&1 || echo "缺失: $c"
   done
   ```

3. **选文件管理器要注意别把刚清掉的依赖带回来。**
   本例中 `dolphin`（KDE）会连带 `baloo`/`kio`/`kcmutils` 等一堆 KDE 库，
   而 `nautilus` 正好是配置里绑定的那个，与 GTK 主题也一致 —— 优先选**配置已经指向的**。
