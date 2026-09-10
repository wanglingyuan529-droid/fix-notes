# 系统

系统安装、磁盘分区、引导、双系统相关排障。

| 日期 | 主题 | 一句话结论 |
|---|---|---|
| 2026-09-08 | [Arch + Win11 双系统安装（ext4 缩容踩坑）](2026-09-08_arch-win11-dualboot-ext4-shrink.md) | resize2fs 用 GiB、parted 用十进制 GB，缩容顺序不能反且每步要验证 |
| 2026-09-09 | [GLava 圆形频谱 · Hyprland 配置与排障全程](2026-09-09_glava-hyprland-visualizer.md) | 自定义波形 shader + hyprwinwrap 垫底桌面层 + 音频源切换要重启；Hyprland Lua 配置改前备份、整文件写 |
| 2026-09-08 | [蓝牙耳机 DM02 突然连不上](2026-09-08_bluetooth-dm02-reconnect.md) | 被 remove 过的设备必须重新扫描发现+配对，别急着怀疑扫描/内核坏了 |
| 2026-09-10 | [fcitx5 5.1 主题不生效（配置文件格式变更）](2026-09-10_fcitx5-theme-not-applied.md) | 5.1.x 的 classicui.conf 改扁平键值、无 [Appearance] 段；用 D-Bus SetConfig 让 fcitx5 自己写盘最稳 |