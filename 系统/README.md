# 系统

系统安装、磁盘分区、引导、双系统相关排障。

| 日期 | 主题 | 一句话结论 |
|---|---|---|
| 2026-09-08 | [Arch + Win11 双系统安装（ext4 缩容踩坑）](2026-09-08_arch-win11-dualboot-ext4-shrink.md) | resize2fs 用 GiB、parted 用十进制 GB，缩容顺序不能反且每步要验证 |
| 2026-09-09 | [GLava 圆形频谱 · Hyprland 配置与排障全程](2026-09-09_glava-hyprland-visualizer.md) | 自定义波形 shader + hyprwinwrap 垫底桌面层 + 音频源切换要重启；Hyprland Lua 配置改前备份、整文件写 |
| 2026-09-08 | [蓝牙耳机 DM02 突然连不上](2026-09-08_bluetooth-dm02-reconnect.md) | 被 remove 过的设备必须重新扫描发现+配对，别急着怀疑扫描/内核坏了 |
| 2026-09-10 | [fcitx5 5.1 主题不生效（配置文件格式变更）](2026-09-10_fcitx5-theme-not-applied.md) | 5.1.x 的 classicui.conf 改扁平键值、无 [Appearance] 段；用 D-Bus SetConfig 让 fcitx5 自己写盘最稳 |
| 2026-09-16 | [双系统蓝牙冲突：Windows 用过耳机后 Linux 连不上](2026-09-16_bluetooth-dualboot-linkkey.md) | 双系统共用适配器但各存 link key，耳机端只认一份；重配是兜底，bt-dualboot 共享 key 是治本 |
| 2026-10-13 | [卸载 KDE Plasma 与 Hyprland：五个会把系统搞坏的坑](2026-10-13_kde-hyprland-removal-five-pitfalls.md) | `-Rns plasma-meta` 会连带删掉 `pipewire-pulse` 导致系统静默无声；虚拟依赖要先 `--asexplicit` 指定提供者 |
| 2026-10-13 | [卸载 waybar 误伤共享依赖：壁纸快捷键、下拉终端、剪贴板历史集体失效](2026-10-13_waybar-removal-broke-shared-deps.md) | `-Rns` 前必须 `--print` 看级联清单；旧组件的依赖可能被新环境复用，先反查反向依赖 |
| 2026-10-13 | [快捷键绑定的程序从未安装：nautilus 缺失导致 Mod+E 无反应](2026-10-13_nautilus-missing-mod-e-dead.md) | niri 的 `spawn` 失败是静默的；快捷键没反应先手工执行绑定命令 |
| 2026-10-15 | [重启 logind 毁掉整个图形会话](2026-10-15_logind-restart-killed-session.md) | `systemctl restart systemd-logind` 会销毁该用户所有会话；配置类改动不需要重启服务，验证优先用只读接口 |
| 2026-10-15 | [所有 Mod 快捷键集体失效：Win 键被键盘硬件锁定](2026-10-15_win-key-locked-diagnosis.md) | 所有 Mod 组合键同时失效时先怀疑修饰键没发出来；`evtest` 抓原始事件流是唯一可靠判据，检查顺序要从硬件往配置走 |
| 2026-10-17 | [pkill -f 把自己杀了](2026-10-17_pkill-self-match-killed-own-shell.md) | `pkill -f` 匹配完整命令行，调用者自身命令行含该串就会自杀；改用 `[y]ay` 方括号技巧或先 `pgrep` 取 PID 再 kill；破坏性操作与善后清理别绑在同一条命令 |
