# yay 构建成功却安装失败：无 tty 环境下 sudo 拿不到密码

> 日期：2026-10-17 · 环境：Arch Linux (x86_64) · pacman 7.1.0 · 目标：visual-studio-code-bin 1.141.1 → 1.142.0

## 现象

在无终端的自动化环境里跑 `yay -S --noconfirm visual-studio-code-bin`：

- 构建阶段**完全正常**：下载源码 → 校验 sha256 → 打包 → `Finished making: visual-studio-code-bin 1.142.0-1`
- 紧接着安装阶段整体失败：

```
sudo: a terminal is required to read the password; either use the -S option to read from standard input
      or configure an askpass helper
sudo: a password is required
 -> error installing: [.../visual-studio-code-bin-1.142.0-1-x86_64.pkg.tar.zst] - exit status 1
```

报错行指向的偏偏是**已经构建好的那个包**，极易误判成"构建失败"。

## 根因

AUR 助手的安装是**两段式**，且提权方式不可配置：

1. **构建段**：`makepkg`（普通用户 + fakeroot），产出 `.pkg.tar.zst`
2. **安装段**：调 `sudo pacman -U <刚产出的包>` —— yay 调的是**裸 `sudo`**（不带 `-S`、不带 `-A`）

于是：

- 没有 tty 时，sudo 既不能从终端读密码，也没有 askpass helper → 直接失败
- `echo pw | sudo -S -v` 预刷时间戳**也救不了**：Arch 默认 `timestamp_type=tty`，时间戳与 tty 绑定，在没有 tty 的不同调用之间**不持久**。每次新调用 yay 都会重新撞上密码提示
- `yay -Syu` 会更早失败在**刷新数据库**那一步（`sudo pacman -Sy` 就挂了，报 `error refreshing databases`），此时连 AUR 包都还没开始处理 —— 错误信息里的阶段名是重要线索

## 排查

先分清错误落在哪一段，别被"error installing + 包路径"骗成构建问题：

```bash
# 构建产物到底有没有？
ls -lh ~/.cache/yay/<pkg>/<pkg>-<ver>-x86_64.pkg.tar.zst

# 当前 sudo 是否需要密码（-n = 非交互）
sudo -n true && echo "凭据有效" || echo "需要密码"
```

产物在、`sudo -n` 却失败 → 纯提权问题，**不需要重新构建**。

## 步骤

### 解法 A（推荐）：直接装它已经构建好的产物

构建段和安装段是分离的，产物完全有效：

```bash
sudo pacman -U --noconfirm ~/.cache/yay/<pkg>/<pkg>-<ver>-x86_64.pkg.tar.zst
```

在自己能控制提权的 shell 里执行（`echo pw | sudo -S ...` 或交互终端），一次装完。

### 解法 B：临时 NOPASSWD，跑完立即撤销

需要连着跑多个 yay 操作时才值得：

```bash
# 1) 加临时规则（00- 前缀保证最先加载）
sudo bash -c 'echo "iris ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/00-temp-yay; \
              chmod 0440 /etc/sudoers.d/00-temp-yay'
sudo visudo -c          # 必须校验语法：sudoers 写坏会让 sudo 全面失效
sudo -n true            # 确认规则已生效

# 2) 跑需要提权的操作
yay -Syu --noconfirm

# 3) 撤销 + 反向验证（关键一步）
sudo rm -f /etc/sudoers.d/00-temp-yay
sudo -k                 # 清掉时间戳缓存，否则下一步会假性通过
sudo -n true && echo "⚠️ 仍免密！" || echo "✅ 已恢复需要密码"
```

## 坑

1. **`rm` 成功 ≠ 规则失效**：删完必须 `sudo -k` 再 `sudo -n true` 反向验证。不清时间戳的话 `sudo -n` 会因缓存而通过，给出**假阳性**——以为撤销了、其实没撤。
2. **别用普通用户去 `ls /etc/sudoers.d/`**：该目录是 `drwxr-x--- root root`，普通用户读不了，`ls` 直接失败。若写成 `ls ... && echo "还在" || echo "已删除"`，权限错误会被 `||` 吞掉，打印出**错误的"已删除"**。要用 `sudo ls`。
3. **构建产物别删**：那个 313MB 的包重建要重下源码，白等一轮。
4. **`yay -Sc` 同样会撞这个坑**：清理缓存也走 sudo，在无 tty 下一样失败。本次实测：`yay -Sc` 报同一个错，pacman 缓存清了 1G，而 yay 自己的 4G **一个字节没动**。此时直接删构建目录即可：

```bash
rm -rf ~/.cache/yay/<pkg>/     # 只删构建目录，保留 vcs.json / completion.cache 等元数据
```

## 验证

```bash
pacman -Q <pkg>                                   # 版本已更新
ldd /usr/share/code/code | grep -c 'not found'    # 0 = 动态库依赖完整
```

## 教训

1. **"构建成功 / 安装失败"是个必须分开看的组合**：错误信息里的包路径是**产物**，不是失败原因。先问"产物在哪、提权在哪一步断的"。
2. **凡是要 sudo 的自动化，都要显式设计提权路径**：`-S`（读 stdin）/ `-A`（askpass）/ 交互 tty，三选一，没有第四种。裸 `sudo` 在无 tty 环境里必然失败——这不是偶发故障，是设计缺口。
3. **`timestamp_type=tty` 是无 tty 自动化里最容易低估的变量**：它让"我刚才 sudo 过"这个假设在跨进程时不再成立。
