# Tor 浏览器在国内网络环境的安装排障

> 日期：2026-09-07 · 更新：2026-09-12 · 环境：Arch Linux · 状态：✅ 已解决

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

## 2026-09-12 解决记录

**安装成功**（走本机 FlClash 代理 `127.0.0.1:7890`，注意不是之前 GNOME 的 7892）：

1. **下载**：代理下载官方 tarball `tor-browser-linux-x86_64-15.0.22.tar.xz`（138MB，
   HTTP 200 且 Content-Length 与官方一致）。
2. **GPG 校验通过**：用 `~/.local/share/torbrowser/gnupg_homedir` 密钥环里的官方公钥
   （指纹 `EF6E286DDA85EA2A4BA7DE684E2C6E8793298290`）验签成功。
3. **解压安装**：解压到 `~/.local/share/torbrowser/tbb/x86_64/`，顶层目录
   `tor-browser/` 与 launcher 期望路径完全匹配（`start-tor-browser.desktop` 存在即视为已装）。
4. **标记已安装**：`~/.config/torbrowser/settings.json` 改 `"installed": true`。
5. **快捷启动**：创建 `~/.local/bin/tor-browser`——必须**先 `cd` 到 tor-browser 目录**
   再执行 `./start-tor-browser.desktop`，脚本内部用相对路径，直接调会报
   `env: "./Browser/execdesktop": 没有那个文件或目录`。

**连接排障**（浏览器能开、Tor 起不来）：

1. 直连失败 → 从 `https://bridges.torproject.org/bridges?transport=obfs4`（走代理）申请到
   官方 obfs4 网桥，写入 `Browser/TorBrowser/Data/Tor/torrc`：
   ```
   UseBridges 1
   Bridge obfs4 <IP>:<端口> <指纹> cert=... iat-mode=0
   ```
   `tor --verify-config` 验证配置有效，但实际仍连不上。
2. GUI 连接页换内置 Snowflake 网桥，也不行。
3. **GitHub 网桥列表备选**（最终没用到）：
   - `scriptzteam/Tor-Bridges-Collector-v2`：`bridges/obfs4_tested.txt` 229 条，
     本机 TCP 快筛仅 **36 条可达**（其余被墙），webtunnel 全是 IPv6 本机无 IPv6 不可用；
   - `center2055/OnionHop-Bridges-Collector`：每小时收集并 TCP 测试；
   - 实测方法：对可达的桥逐个起临时 tor 实例（`UseBridges 1` + 单桥 + `SocksPort 0`），
     看日志 bootstrap 是否到 100%。
4. **最终解决：不设置代理（浏览器直连）**，成功连入——"开着代理反而连不上"。

**网桥（Bridge）是什么**：Tor 的"秘密入口"。公开中继节点地址被墙全封，网桥是官方
不公开分发的隐藏入口，obfs4 把流量混淆成普通 HTTPS 规避识别。

## 最终状态

✅ 已解决（2026-09-12）：Tor Browser 15.0.22 安装完成，应用菜单 / `tor-browser` 启动。
连接需保持系统代理关闭；obfs4 网桥配置留在 torrc 中备用。
