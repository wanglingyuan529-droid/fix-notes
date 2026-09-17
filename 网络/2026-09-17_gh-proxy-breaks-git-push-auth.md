# gh-proxy 代理重写破坏了 git push 认证：凭据管理器认不出 GitHub，退化成密码框

> 日期：2026-09-17 · 环境：Windows 11 (x86_64) · git for Windows（自带 GCM）· 仓库 coding-diary

## 现象

push 报认证失败：

```
remote: Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed for
  'https://gh-proxy.com/https://github.com/wanglingyuan529-droid/coding-diary.git/'
```

同时凭据管理器弹出来的是一个**通用的"用户名 + 密码"对话框**（不是 GitHub 的浏览器登录页），
无论输什么都失败。

## 根因

**代理重写规则把所有操作（含 push）都改走了代理，而凭据管理器不认识代理的 host。**

1. 为了绕过 github.com 直连不稳定，全局配了：
   ```
   url."https://gh-proxy.com/https://github.com/".insteadOf = "https://github.com/"
   ```
   这条规则对 **fetch 和 push 一律生效** —— 推送也被重写成 `gh-proxy.com/...`。

2. Git Credential Manager 靠**真实 host** 判断该走哪家 OAuth 流程。
   host 变成 `gh-proxy.com` 后它识别不出，日志里直接说了：
   ```
   warning: auto-detection of host provider took too long (>2000ms)
   warning: see https://aka.ms/gcm/autodetect for more information.
   ```
   于是退化到**通用用户名/密码框**。

3. GitHub **早已禁用密码认证**（只接受 PAT 或 OAuth token）→ 无论输什么都报
   `Password authentication is not supported`。

> 注意 `fatal` 里那个 URL —— `https://gh-proxy.com/https://github.com/...` 就是最直接的线索：
> 推送确实走了代理。

## 试错（记录一个无效做法）

想用命令行临时压住全局规则：

```powershell
git -c 'url.https://gh-proxy.com/https://github.com/.insteadOf=' push origin main
# ❌ 无效：fatal 里的 URL 仍然是 gh-proxy，空值压不住全局重写
```

## 修复

给 push 单独加一条**恒等重写**（顺序很重要：git 在 push 时 **pushInsteadOf 优先于 insteadOf**）：

```powershell
git config --global --add url.https://github.com/.pushInsteadOf https://github.com/
```

效果：**拉取走代理（稳定），推送直连 github.com（GCM 认得 GitHub，走浏览器 OAuth）**。

配置后应看到两条规则并存：

```
url.https://gh-proxy.com/https://github.com/.insteadof  https://github.com/
url.https://github.com/.pushinsteadof                    https://github.com/
```

随后正常 `git push origin main` 会弹出**浏览器 GitHub 授权页**，登录一次 token 即持久化。

## 坑

1. **代理规则不只影响下载**：为"clone/fetch 加速"加的 insteadOf，会把 push 一起改道。
2. **凭据管理器的 OAuth 依赖真实 host**：任何 URL 改写都可能让 OAuth 退化成密码认证，
   而 GitHub 不支持密码 → 表现为"输对密码也失败"，极具误导性。
3. **弹窗类型是重要信号**：Web OAuth 会开浏览器；弹出的是应用内用户名/密码框 → host 没被识别。
4. **空值 `-c` 覆盖全局 `insteadOf` 无效**（本机实测），别在这条路上浪费时间。
5. **首次走对 host 的 OAuth 只做一次**：token 存进 Windows 凭据管理器后，后续 push 免登录。

## 验证

```powershell
git push origin main
# To https://github.com/wanglingyuan529-droid/coding-diary.git
#    b93e3c8..2558219  main -> main        ← exit 0，URL 已是真实 github.com

git status --short --branch
# ## main...origin/main                    ← 不再有 ahead，与远程同步
```

再用 GitHub API 核对提交确实落到远端（作者邮箱为 noreply，归属账号）。

## 教训

1. **优化读取路径时，先问一句"写操作会怎样"**：代理/镜像规则的副作用面比想象的大。
2. **拉取与推送可以分开治理**：`insteadOf` 管读、`pushInsteadOf` 管写——
   一个恒等重写就能让两者各走各的路，不必二选一。
3. **认证失败先看 URL，再看弹窗形式**：URL 暴露是否被改写，弹窗形式暴露凭据管理器是否识别了 host。
4. **把有效方案写进技能文档，并注明无效做法**：这次把"空值绕过无效"也记下来，避免以后重复试错。
