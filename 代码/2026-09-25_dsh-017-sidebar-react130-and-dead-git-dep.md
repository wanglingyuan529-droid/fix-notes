# DSH 0.1.7 升级后侧边栏 React #130 + 一个已消失的 git 仓库依赖卡死 pnpm

> 日期：2026-09-25 · 环境：Windows 11 (x86_64) · DSH 0.1.5-rc.2 → 0.1.7-rc.1 · profile: web

## 现象

1. 浏览器控制台：`dsh-better-sidebar: Minified React error #130`（React "Element type is invalid... got: undefined"），侧边栏异常。
2. 想升级插件时 `dsh plugin add` 失败：卡在 git 依赖 `github:yq04/dsh-git-remotes`，报
   `remote: Invalid username or token. Password authentication is not supported`。

## 根因

**两个独立问题，叠加在同一台机器上：**

### 问题一：React #130 = 插件 peer 收窄在旧核心，引用了被改名的导出

- 已装 `dsh-better-sidebar@0.19.1`（2026-09-11 发布，peer 全部 `^0.1.5-rc.1`）。
- 核心 9/24 升到 **0.1.7-rc.1**：0.19.1 客户端 bundle 里引用的某组件在 0.1.7 里解析成 `undefined`，
  `React.createElement(undefined)` → error #130。
- npm 上 **0.21.1**（2026-09-23 发布）的 peer 全部为 `^0.1.7-rc.1` —— **作者专为 0.1.7 出的兼容版**。

### 问题二：git 依赖的仓库从 GitHub 消失 → 任何 pnpm 安装都失败

- package.json 里 `"dsh-git-remotes": "github:yq04/dsh-git-remotes"`。
- 该仓库已被作者删除/转私有（GitHub API **404**，作者账号仓库列表也为空）。
- pnpm 每次 install 都要重解析 git 依赖 → 对已消失的仓库 `ls-remote` → GitHub 拒绝
  （走 gh-proxy 时表现为 "Invalid username or token"，实际是仓库不存在被当作需认证处理）。
- 排查确认：gh-proxy git 通道本身正常（对存在的仓库 omdsh-dev 秒回 HEAD），是**仓库没了**。

## 排查

```powershell
# 1. 确认插件版本 vs npm 最新 + peer 范围
node -p "require('.../dsh-better-sidebar/package.json').version"          # 0.19.1
npm view dsh-better-sidebar@0.21.1 peerDependencies                       # 全是 ^0.1.7-rc.1

# 2. 区分"代理坏了"还是"仓库没了"（关键一步）
git ls-remote https://gh-proxy.com/https://github.com/omdsh-dev/dsh-tool-calculator.git HEAD   # 通 → 代理没问题
curl -s -o NUL -w '%{http_code}' https://api.github.com/repos/yq04/dsh-git-remotes             # 404 → 仓库没了
```

## 修复

### 1. 移除已消失的仓库依赖

从 `profiles/web/package.json` 的 `dependencies` 和 `dsh.profile.bundles` 两处删掉 `dsh-git-remotes`。
（该插件"Git 远端管理"本身无替代必要；作者删库=停止维护，留着只会让每次安装都失败。）

### 2. 升级 dsh-better-sidebar 到 0.21.1

```powershell
dsh plugin --profile web add dsh-better-sidebar@0.21.1
```

### 3. 体检其余插件的 peer 范围（Node 读 UTF-8，别用 PS 5.1 ConvertFrom-Json）

扫 30 个 bundle 的 `peerDependencies["@deepseek-ai/dsh-*"]`：
- 收窄在 `^0.1.5-rc.1` 且不含 0.1.7 的 → 高风险（就是本次的 dsh-better-sidebar）
- `rc.6` 时代的宽范围（`^0.1.0-rc.6`、`>=0.0.1-rc.1 <0.2.0`）→ 通常兼容，无症状不折腾
- 范围里含 `0.1.7-0` / `0.1.7-rc` 的 → 安全

## 坑

1. **React #130 的 `undefined` 是"符号解析失败"的信号**：优先怀疑「插件客户端 bundle 按旧核心编译、引用的组件在新核心改名/移除」。查插件 npm 最新版的 peer 范围即可确认——作者通常会在核心升级后补兼容版。
2. **git 依赖的仓库可能"蒸发"**：`github:owner/repo` 依赖没有版本锁死，作者删库/强推后 pnpm 全挂。
   排查顺序：先验证**代理本身通不通**（找一个确定存在的仓库试），再查**目标仓库存在性**（GitHub API 状态码）。
3. **错误文案会骗人**：仓库不存在时通过 gh-proxy 表现为 "Invalid username or token"（GitHub 对不存在的私有化处理），
   看起来像凭据问题，实际是 404 —— 别去折腾 GCM/凭据（清掉 gh-proxy.com 的坏凭据也没用）。
4. **DSH 升级会静默替换设置架构**（0.1.5 的 settings.yaml → 0.1.7 的 profile Cordis patch），
   且只导入一部分 section —— 升级大版本后要核对 `cordis.patch.yml` 与 `settings.yaml.imported`。

## 验证

```powershell
dsh --profile web --dump-config        # 无报错；含 better-sidebar 行；不含 git-remotes
node -p "require('.../dsh-better-sidebar/package.json').version"   # 0.21.1
```

重启 dsh web + 硬刷新后，侧边栏正常、控制台无 React #130。

## 教训

1. **插件"能装"≠"兼容"**：peer 范围只要语义满足就会装上去，但客户端 bundle 是按编译时的核心生成的。
   核心跨版本升级后，收窄 peer 的插件是最容易炸的一批。
2. **依赖体检要有"活体检查"**：静态范围扫描只能给风险清单；`git ls-remote` 一个对照仓库 + GitHub API
   状态码才是"到底通不通"的实锤。
3. **遇到诡异的认证错误先怀疑"对象不存在"**：Invalid username/token 这类文案，
   在代理/镜像场景下经常是 404 或 403 的伪装。
