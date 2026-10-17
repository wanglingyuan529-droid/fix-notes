# GitHub Release 大文件下载：镜像测速会骗人，断点续传才是保命符

> 日期：2026-10-17 · 环境：Arch Linux · 对象：folia-major-bin AUR 构建，需从 GitHub Releases 拉 162MB
> 关联：[2026-09-10 Folia 播放器 AUR 安装](2026-09-10_folia-aur-install.md)（同源问题的第一次记录）

## 现象

AUR 构建（`yay -Syu`）在下载阶段几乎卡死：

```
-> Downloading folia-major-0.7.20-linux-x64.tar.gz...
  0 162.0M   0  1.68M   0  0  17911   0  2:38:05        ← 约 17KB/s，预计 2 小时 38 分
```

这与 9 月那次（27KB/s）是同一个问题。中途中断一次，已经下好的 33MB 眼看要白费。

## 根因

三层，缺任何一层都不会这么难受：

1. **GitHub Releases 在国内直连不稳定**：带宽被压到几十 KB/s 量级（9 月已记录）
2. **免费镜像会限速且剧烈波动，短时测速无法代表持续速度**：
   - 测速：`ghfast.top` 1.4MB/s
   - 实际持续：掉到 70KB/s
   - 隔几分钟复测同一镜像：又是 597KB/s，而这次真下起来爬到了 3.4MB/s
3. **curl 默认不具备续传能力**：中断即从 0 开始，慢速环境里每次断线都全额重来

## 排查

对候选镜像做短时测速。**它只能用来"排除"，不能用来"决定"**：

```bash
for m in "https://ghfast.top" "https://gh-proxy.com" "https://ghproxy.net" "https://hub.gitmirror.com"; do
  printf "%-32s " "$m"
  timeout 16 curl -o /dev/null -sL -w "HTTP %{http_code}  %{speed_download} B/s\n" \
    --max-time 10 "$m/https://github.com/<owner>/<repo>/releases/download/v<ver>/<file>"
done
```

本次结果：

| 镜像 | 测速 |
|---|---|
| `ghfast.top` | 597 KB/s（实测下载时峰值 3.4MB/s） |
| `gh-proxy.com` | 110 KB/s |
| `ghproxy.net` | 52 KB/s |
| `hub.gitmirror.com` | 失败（HTTP 000） |

## 步骤

### 1. 已经在下的部分**不要删**

中断留下的半成品就是断点续传的本钱。急着 `rm` 等于把已下的 33MB 一起扔掉。

### 2. 断点续传

```bash
curl -L -C - --retry 3 --retry-delay 2 \
  -o folia-major-0.7.20-linux-x64.tar.gz \
  "https://ghfast.top/https://github.com/chthollyphile/folia-major/releases/download/v0.7.20/folia-major-0.7.20-linux-x64.tar.gz"
```

`-C -` = 从文件当前长度继续。本次 **33MB 起步，1 分 29 秒拉完剩余 129.5MB**（平均 1.44MB/s，峰值 3.4MB/s）。

### 3. 放进 makepkg 构建目录，让它跳过下载

```bash
mv <下载好的文件> ~/.cache/yay/<pkg>/
cd ~/.cache/yay/<pkg> && makepkg -s --noconfirm
```

makepkg 发现同名源码已存在，直接进入打包阶段。

### 4. **自己**校验完整性（重要）

```bash
tar -tzf folia-major-0.7.20-linux-x64.tar.gz > /dev/null && echo "完整"
```

因为该 PKGBUILD 里 `sha256sums` 对 tar.gz 写的是 **`SKIP`** —— 官方校验被跳过了，**没有任何自动机制保证你下到的文件是完整的**。

## 坑

1. **镜像测速只能证伪，不能证实**：1.4MB/s 的测速后面跟着 70KB/s 的实际下载。估时要按**最坏情况**算。
2. **`-C -` 需要服务端支持 Range 请求**：不支持时会静默从头下载。可用 `curl -v` 看有没有发出 `Range:` 头。
3. **`sha256sums=('SKIP' ...)` 意味着零官方校验**：必须自己 `tar -tzf` 或至少比对文件大小，否则可能把**被截断的包**打进系统。
4. **测速时不要并发下载**：多个 curl 抢同一条链路，测出来的数字全是错的。
5. **9 月那次的解法（预放源码包）和这次是同一招**，但这次多了"保留半成品 + 续传"这一环。结论更新为：**慢速环境下，能续传比找到快镜像更重要**。

## 验证

```bash
ls -lh <file>                                # 大小与上一版可比（本次 162M，对上 0.7.19 的 162M）
tar -tzf <file> > /dev/null && echo "完整"    # 压缩包结构可读
pacman -Q folia-major-bin                    # 装好后版本正确
```

## 教训

1. **慢速不可怕，重来才可怕**：大文件下载要按"可续传"来设计——`-C -`、保留半成品、别急着删。
2. **测速是瞬时快照，不是承诺**：网络环境里唯一可靠的组合是"能续传 + 能校验"。
3. **没有校验的地方必须自己补校验**：`SKIP` 省掉的是构建者的麻烦，转嫁出来的是使用者的风险。
