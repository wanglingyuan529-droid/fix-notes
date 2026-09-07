# fix-notes

个人报错排障流水账：遇到什么问题、怎么定位、怎么解决、下次怎么避免。
每条记录一个文件，`notes/日期_主题.md`，README 维护索引。

## 索引

| 日期 | 主题 | 一句话结论 |
|---|---|---|
| 2026-09-06 | [VideoToNo Linux 安装](notes/2026-09-06_videotono-linux-install.md) | Arch 上源码装通，Python 必须用 3.11 虚拟环境 |
| 2026-09-06 | [B站扫码登录 Linux 失效](notes/2026-09-06_videotono-bilibili-login.md) | 上游没写 Linux 浏览器路径，补路径 + PATH 兜底修复 |
| 2026-09-07 | [Tor 浏览器网络封锁排查](notes/2026-09-07_tor-browser-network-blocked.md) | 官网被墙 + 代理节点被 Tor 拉黑，双重墙无解 |
| 2026-09-07 | [校园网 E63073 场景拒绝](notes/2026-09-07_campus-network-e63073.md) | 已解决：学号注册时 3/8 写反，E63073 只是模糊的凭据错误文案 |

## 记录规则

1. 每条记录一个文件：`notes/YYYY-MM-DD_主题.md`
2. 统一结构：**现象 → 根因 → 修复 → 验证 → 教训**
3. 结论要能一句话说清（写给未来的自己看）
4. 新增记录后在 README 索引表加一行
