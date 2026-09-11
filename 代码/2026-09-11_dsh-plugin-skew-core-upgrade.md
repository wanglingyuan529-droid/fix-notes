# DSH 插件版本歪斜导致 web 崩溃：核心 0.1.5 升级与三插件适配

> 日期：2026-09-11 · 环境：Arch Linux (x86_64) · 版本：dsh 0.1.1-rc.2 → 0.1.5-rc.2

## 现象

`dsh web` 启动即崩：ESM 命名导入缺失（`import { SessionLogOffset } from "@deepseek-ai/dsh-session"` 找不到符号），
整棵插件树加载失败，3080 不监听。

起因是 dshmarket 批量更新插件时把 `dsh-better-sidebar` 从 0.17.1 升到 0.18.1，而 0.18.1 的 peer 声明为
`@deepseek-ai/dsh-session: ^0.1.2-rc.1`——核心还停在 0.1.1-rc.2，**插件跑在了核心前面**。

## 环境

| 项 | 值 |
|---|---|
| 系统 | Arch Linux (x86_64) |
| 核心 | `~/.local/lib/node_modules/@deepseek-ai/dsh`（npm 全局） |
| profile | `~/.dsh/profiles/web`（pnpm） |
| pnpm 设置 | `nodeLinker: hoisted` · `autoInstallPeers: false` |

`autoInstallPeers: false` 是这次的关键：peer 不满足时 pnpm 不会自动装一份独立副本，
插件 import 一律解析到核心自带的旧包，所以缺符号直接硬错。

## 步骤

### 1. 先降级插件恢复可用

```bash
dsh plugin --profile web add dsh-better-sidebar@0.17.1 --registry=https://registry.npmmirror.com
```

### 2. 用导出差集扫描代替「崩了才知道」

写了个体检脚本（`check-plugin-core-compat.py`）对比「当前核心」与「候选新核心」的
`@deepseek-ai/*` 导出集合，再扫已装插件的 `import { ... }`，直接列出会缺导出的插件。
把候选核心装到临时 prefix 即可对比，不必真升：

```bash
rm -rf /tmp/dshnew && mkdir -p /tmp/dshnew
npm install --prefix /tmp/dshnew @deepseek-ai/dsh@0.1.5-rc.2 \
  --registry=https://registry.npmmirror.com --no-audit --no-fund
```

扫描结果：核心 0.1.2-rc.1 起**移除了 `dsh-settings` 的三个导出**
（`settingsNamespace`、`installSettingsSection`、`deepEqualJson`，后者挪到新包 `dsh-util-values`），
受影响插件三个。

### 3. 逐个适配（核心升到 0.1.5-rc.2 之前做完）

| 插件 | 用法 | 处置 |
|---|---|---|
| `dsh-web-search-bing`（本地插件） | `settingsNamespace` + `installSettingsSection` | 源码内联这两个函数（后者约 25 行，旧实现可直接照搬） |
| `@ha-na-bi/dsh-client-ui-custom` | `settingsNamespace`（对常量做校验，恒等） | clone 到 `~/.dsh/plugins/` 本地化，依赖改 `file:`，直传常量 |
| `dsh-better-sidebar` 0.18.1 | 已弃用旧 API（只用 `SettingsConflictError` + `SessionLogOffset`） | 无需改，升回 0.18.1 |
| `dshmarket` 1.37.0 → 1.45.1 | 上游已适配，但 1.37 仍用旧导出 | 升到 1.45.1 |

注意 `dsh-better-sidebar` 0.18.1 本来就是为新核心写的——**「升核心要修三个插件」的判断在核实 0.18.1 源码后变成只需修两个**。

### 4. 升级核心并验证

```bash
npm install -g @deepseek-ai/dsh@0.1.5-rc.2 --registry=https://registry.npmmirror.com
dsh --profile web --dump-config      # 退出码 0 + stderr 空
```

### 5. 核心自带 file-upload 行与第三方插件撞 id

0.1.5 的 `dsh-web-app` bundle 自带一行 `id: file-upload`（官方浏览器上传基础服务），
而第三方 `dsh-file-upload` 自带的 `cordis.patch.yml` 也用同一个 id，
内核报 `duplicate loader entry id: file-upload` 拒绝启动。

**修法：改插件自带 patch 的行 id**（`node_modules/dsh-file-upload/cordis.patch.yml` → `id: file-upload-ext`）。
行的 `name` 才是模块解析依据，`id` 只是树里的键，改名不影响功能。

## 坑

1. **`dsh plugin` 增删会重写 profile 的 bundles 列表**：先前「把插件从 bundles 摘出去」的修法，
   被一次 `dsh plugin remove` 重写撤销，重复 id 复发。**修复要放在不会被别的工具重写的层**——
   改插件自带的 patch 优于改 profile 的组合状态。
2. **npmmirror 的 tarball 302 到 `cdn.npmmirror.com` 时不可达**（SSL 连接失败），
   pnpm 的 fetch 会直接失败；而 npm 走同一镜像可以成功下载。
   绕法：临时用 HTTP 代理（`HTTPS_PROXY=http://127.0.0.1:7890`）+ 官方源。
3. **`dsh plugin add` 网络失败时不会改动 package.json**，可放心重试。
4. **file: 依赖的副本同步**：改了本地插件源码后，`pnpm install` 未必刷新 `node_modules` 里的副本
   （pnpm 认为 file: 源未变化），需确认内容或手动同步。

## 验证

```bash
dsh --profile web --dump-config       # EXIT 0，stderr 0 字节
dsh --profile headless --dump-config  # 同样通过（两 profile 共用全局核心）
```

- 全量 import 扫描（含单双引号）：待修插件对 0.1.5-rc.2 的**缺导出数 = 0**
- `dump-config` 中确认两行并存：`id: file-upload`（核心）+ `id: file-upload-ext`（第三方），全树无重复 id
- dshmarket `check` 端点：185 行 0 error、0 orphan、0 duplicate、0 multiVersion
- web 真实启动：3080 监听、`curl /` 正常、`/plugins/<插件>/client.js` 200

## 教训

1. **插件与核心的版本关系要双向看**：插件 peer 范围宽（`^0.1.0-rc.8`）会给核心留出「看起来能装」的空间，
   实际用的是比它预期更旧的核心。升插件前先确认核心版本落在 peer 范围内。
2. **静态导出差集扫描是划算的投资**：一次脚本扫描（几分钟）替代「装上去崩了再回滚」，
   尤其适合 rc 阶段频繁变动的核心。
3. **修复写在哪一层决定了它活多久**：profile 的组合状态会被任何 `dsh plugin` 操作重写；
   插件自带文件、独立配置才是稳定的落点。
4. **第三方托管的模型/包同理会滞后**：核心生态里的版本对齐问题，最终都要靠「谁提供、谁负责」来判断修复位置。
