# Folia 播放器 AUR 安装（GitHub 慢速换镜像）

日期：2026-09-10 · 环境：Arch Linux (x86_64) + yay 13.0.1 + electron43/ffmpeg/numpy 已就绪

## 背景与目标

安装 Folia（folia-major，歌词动画播放器，Electron 桌面端，v0.7.7）——走 AUR 源码构建，装完由 pacman 统一管理更新。

## 最终成果（结论先行）

- **folia-major 0.7.7-1 已装入系统**，`/usr/bin/folia-major` 可执行，桌面入口（.desktop）已注册到应用菜单
- 安装体积 155.22 MiB，构建全程约 7 分钟
- 后续升级只需 `yay -S folia-major`

## 前置检查（一步都不能省）

| 检查项 | 结果 |
|---|---|
| AUR 包 `folia-major`（v0.7.7-1，与官方最新 Release 同步） | 存在，维护者 zxp19821005 |
| 运行时依赖 electron43 / python / python-numpy / python-psutil / ffmpeg | 本机已全装 ✓ |
| 构建依赖（makedepends）npm / nvm / git / curl / jq | **nvm 缺失** → `pacman -S nvm` 补装 |

## 排障历程（现象 → 根因 → 修复）

### 1. GitHub 直连下载源码慢到不可用
- **现象**：yay 构建时 `folia-major-0.7.7.tar.gz` 下载速度仅 ~27KB/s。
- **根因**：GitHub 直连（codeload）到本机网络不佳。
- **修复**：停掉任务 → 测速选最快的加速镜像 → 手动下载源码放进 yay 缓存目录 → 重新 yay 构建（makepkg 检测到源码已存在且 SHA-256 匹配，自动跳过下载）。

```bash
# 测速（本次最优：gh-proxy.com，~2.7MB/s）
curl -sL -o /dev/null -w "%{speed_download}\n" \
  "https://gh-proxy.com/https://github.com/chthollyphile/folia-major/archive/refs/tags/v0.7.7.tar.gz"

# 下载到 yay 构建缓存，并用 PKGBUILD 里的 sha256sum 校验
curl -sL -o ~/.cache/yay/folia-major/folia-major-0.7.7.tar.gz \
  "https://gh-proxy.com/https://github.com/chthollyphile/folia-major/archive/refs/tags/v0.7.7.tar.gz"
echo "cf602d61...dac4f93  folia-major-0.7.7.tar.gz" | sha256sum -c -
```

> 经验：yay 下载卡在 GitHub 时，不用改 PKGBUILD 换源——先把源文件手动放进 `~/.cache/yay/<包名>/`，哈希对得上就会跳过下载。

### 2. 构建缺 nvm
- **根因**：PKGBUILD 的 `makedepends` 声明了 nvm，本机没装；其 `_ensure_local_nvm` 会 `source /usr/share/nvm/init-nvm.sh`。
- **修复**：`sudo pacman -S nvm`，重跑 yay 构建即通过。

### 3. yay 后台构建时的 sudo 交互
- **现象**：构建完 pacman 安装那步要 sudo 密码，后台任务里没法交互。
- **修复**：写 SUDO_ASKPASS 脚本 + `sudo -A` 免交互通道，yay 内部调用 sudo 时自动取密码。

## 构建流程备忘（自动，无需干预）

nvm 装 node 24.21.0 → `npm install`（脚本检测 CN IP 自动切 npmmirror）→ vite 打包前端（2.4s）→ electron-builder 26.15.3 用系统 electron43 打包（不重复下载 Electron 二进制）→ makepkg 出包 → pacman 安装 + 更新桌面 MIME 缓存。

## 教训清单（下次避免）

1. **AUR 构建前先核对 makedepends**，缺的提前装，省一轮失败重试。
2. **GitHub 下载慢先测速镜像再动手**，不要干等；手动放好源码包可跳过 yay 下载阶段。
3. **后台跑 yay 时预配 SUDO_ASKPASS**，避免 pacman 步骤卡交互。
4. 装完验证三件套：`pacman -Q` 版本、`/usr/bin/<名>` 存在、`/usr/share/applications/*.desktop` 注册。

## 回滚

```bash
sudo pacman -R folia-major
# 主题皮肤类周边（本次顺带装的）：
sudo pacman -R fcitx5-material-color nvm   # 按需
```
