# 代码

代码 bug、安装部署、开发环境相关排障。

| 日期 | 主题 | 一句话结论 |
|---|---|---|
| 2026-09-06 | [VideoToNo Linux 安装](2026-09-06_videotono-linux-install.md) | Arch 上源码装通，Python 必须用 3.11 虚拟环境 |
| 2026-09-06 | [B站扫码登录 Linux 失效](2026-09-06_videotono-bilibili-login.md) | 上游没写 Linux 浏览器路径，补路径 + PATH 兜底修复 |
| 2026-09-10 | [Folia 播放器 AUR 安装（GitHub 慢速换镜像）](2026-09-10_folia-aur-install.md) | GitHub 直连 27KB/s → 镜像 2.7MB/s 预放源码包跳过下载；补装 nvm 后 yay 构建安装成功 |
| 2026-09-11 | [DSH 插件版本歪斜与核心 0.1.5 升级适配](2026-09-11_dsh-plugin-skew-core-upgrade.md) | 插件 peer 超前核心导致 web 硬崩；导出差集扫描定位三插件适配，id 冲突改成插件自带 patch 才不被重写撤销 |
| 2026-09-11 | [dsh-raw-html 前端渲染补丁在新核心失效](2026-09-11_dsh-raw-html-frontend-patch.md) | 渲染能力靠改 dist 补丁，核心升级即静默失效；本地 React 全绿但浏览器端异常，最终卸载并回落源码显示 |
| 2026-09-17 | [VSCode Java 报 non-project file](2026-09-17_vscode-java-non-project-file-maven.md) | 配置写在了子目录 `.vscode`（工作区设置只读根目录），从未生效；加 pom.xml 让插件递归识别才是解法 |
| 2026-09-17 | [PowerShell 5.1 读写 UTF-8 中文源码导致静默损坏](2026-09-17_powershell-utf8-chinese-corruption.md) | GBK 解码把中文后的闭合引号一起吞掉，编译报"未结束的字符串文字"；改用 Node 显式 utf8 一步做完 |
| 2026-09-17 | [DSH 插件"装了却不生效"的三类静默失败](2026-09-17_dsh-plugin-silent-failures.md) | 被 dshmarket 静默禁用 / 两插件抢同一 UI 面 / 缺 ffmpeg 静默降级；路由 404-405 与 market 日志是关键线索 |
| 2026-09-25 | [DSH 0.1.7 升级后侧边栏 React #130 + 已消失的 git 仓库依赖](2026-09-25_dsh-017-sidebar-react130-and-dead-git-dep.md) | 收窄 peer 的插件引用被改名导出→组件解析成 undefined；升级作者 0.21.1 兼容版；`github:yq04/...` 仓库已删，移除依赖才解封 pnpm |
| 2026-10-17 | [yay 构建成功却安装失败（无 tty 拿不到 sudo 密码）](2026-10-17_yay-no-tty-install-failed.md) | 构建段与安装段是分离的，产物本身有效；装 `~/.cache/yay/<pkg>/` 里的包即可；临时 NOPASSWD 必须 `sudo -k` 后反向验证，`yay -Sc` 同坑 |
| 2026-10-17 | [GitHub Release 大文件下载：镜像测速会骗人](2026-10-17_github-release-large-file-mirror-and-resume.md) | 测速 1.4MB/s、实测 70KB/s；保留半成品 + `curl -C -` 续传，33MB 起步 1 分半拉完；`sha256sums=SKIP` 时必须自己 `tar -tzf` 校验 |
