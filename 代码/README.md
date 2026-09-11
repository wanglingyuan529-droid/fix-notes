# 代码

代码 bug、安装部署、开发环境相关排障。

| 日期 | 主题 | 一句话结论 |
|---|---|---|
| 2026-09-06 | [VideoToNo Linux 安装](2026-09-06_videotono-linux-install.md) | Arch 上源码装通，Python 必须用 3.11 虚拟环境 |
| 2026-09-06 | [B站扫码登录 Linux 失效](2026-09-06_videotono-bilibili-login.md) | 上游没写 Linux 浏览器路径，补路径 + PATH 兜底修复 |
| 2026-09-10 | [Folia 播放器 AUR 安装（GitHub 慢速换镜像）](2026-09-10_folia-aur-install.md) | GitHub 直连 27KB/s → 镜像 2.7MB/s 预放源码包跳过下载；补装 nvm 后 yay 构建安装成功 |
| 2026-09-11 | [DSH 插件版本歪斜与核心 0.1.5 升级适配](2026-09-11_dsh-plugin-skew-core-upgrade.md) | 插件 peer 超前核心导致 web 硬崩；导出差集扫描定位三插件适配，id 冲突改成插件自带 patch 才不被重写撤销 |
| 2026-09-11 | [dsh-raw-html 前端渲染补丁在新核心失效](2026-09-11_dsh-raw-html-frontend-patch.md) | 渲染能力靠改 dist 补丁，核心升级即静默失效；本地 React 全绿但浏览器端异常，最终卸载并回落源码显示 |
