# Tor 浏览器在国内网络环境的安装排障

> 日期：2026-09-07 · 环境：Arch Linux · 状态：未解决（网络环境限制，方案已备好）

## 现象

想装 Tor 浏览器，发现官网根本连不上；排查后发现是「官网被墙 + 代理节点被 Tor 拉黑」双重问题。

## 排查链（按顺序）

1. **本地检查**：系统已有 `tor` 守护进程 + `torbrowser-launcher`，但浏览器本体未下载
   （`~/.local/share/torbrowser/tbb/x86_64` 是空目录，只有 8KB）。
2. **官网直连**：`dist.torproject.org` 连接超时（被墙）。
3. **国内大厂镜像**：阿里/腾讯/交大/USTC 均无 torproject 镜像；**华为云返回 200 但是假象**——
   CDN 门户对任意路径都回 HTML 落地页，验证文件头发现不是 xz。
4. **flathub**：无 Tor 浏览器本体（只有 launcher，下载仍走被墙官网）。
5. **GitHub 镜像仓库**（eyedeekay/torbrowser）：只有老版 Windows 安装包，无 Linux 新版 tarball。
6. **发现本机代理**：GNOME 系统代理 `127.0.0.1:7892`（shell 的 curl 默认不走，要显式 `-x`）。
   走代理后 google/youtube 全部 200（代理正常），但 torproject.org 全家（www/dist/cdn/blog）
   TLS 握手被掐断（`unexpected eof`）→ 判断：**代理节点出口 IP 被 Tor CDN 反滥用策略拉黑**
   （Tor 对数据中心/机场 IP 管控严格）。
7. **换多个节点重试**：仍然全被掐断 → 放弃本次安装。

## 根因

- Tor 官网及全部分发渠道在国内被网络封锁；
- 代理节点出口 IP 段被 Tor 项目反滥用策略拒绝（不同于 Google/YouTube 不禁机房 IP）；
- 官方镜像（NetCologne/EFF/Brave 等）要么只镜像网页不镜像下载文件，要么同样被墙。

## 教训

1. **shell 不走系统代理**：GUI 里开了梯子，curl 一样超时——要显式 `curl -x http://127.0.0.1:端口`。
2. **镜像站会「假 200」**：CDN 门户吞掉一切路径返回 200，必须验文件头（xz = `FD 37 7A 58 5A`）
   或 Content-Type，别信状态码。
3. **Tor 在国内是双重墙**：安装要能访问官网的网络，连接还要网桥（obfs4/webtunnel）。
4. **差分判断**：google 通 + torproject 不通 = 节点被特定站点拉黑，不是代理坏了。

## 备选方案（留档）

拿到能访问 torproject.org 的网络后：代理 + 官网 tarball 下载 → 解压到
`~/.local/share/torbrowser/tbb/x86_64` → `start-tor-browser`，5 分钟可完成。
