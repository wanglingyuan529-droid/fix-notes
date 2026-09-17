# DSH 插件"装了却不生效"的三类静默失败：market 禁用、抢同一 UI 面、缺外部二进制

> 日期：2026-09-17 · 环境：Windows 11 (x86_64) · dsh 0.1.5-rc.2 · profile: web

## 现象

两件看起来无关的事，排查下来是同一类问题——**插件在树里，但功能静默不生效，日志无报错**：

1. 深色皮肤（dream-skin）效果全无，但其宿主路由 `/dream-skin/api` 也不通
2. 壁纸插件（wallpaper-engine）能列出 57 个壁纸，选中后**画面什么都不显示**

## 排查方法（这套手法可复用）

### 1. 用宿主路由判断"插件到底注册了没有"

```powershell
curl.exe -s -o - -w "`nHTTP:%{http_code}" -X POST "http://127.0.0.1:3080/dream-skin/api" `
  -H "Content-Type: application/json" -d '{"method":"get"}'
```

结果 `405`（**空 body**）而不是插件的 JSON 响应 → 说明请求落到了**静态文件兜底**
（`dsh-host-frontend-static` 对非 GET/HEAD 一律 405、GET 未命中 404），即**路由根本没注册**。

> 关键判读：**404/405 + 空 body = 静态兜底**；插件自己的错误一律是带 JSON body 的响应。

### 2. 起一个隔离实例做对照

```powershell
dsh --profile web --port 3099 --no-open
```

新实例同样 405 → 排除"当前进程状态陈旧"，问题在**配置/组合层**，不在运行时。

### 3. 隔离测试插件宿主模块

直接用 Node 导入插件、喂假 ctx 调 `apply()`：`IMPORT OK` + `registered: prefix /dream-skin/api`
→ **插件代码本身完全正常**，矛头转向"它有没有被启动"。

### 4. 查 dshmarket 的状态与日志（真凶在这里）

```powershell
Get-Content "$env:USERPROFILE\.dsh\profiles\web\.dsh-market\state.json"
# {"disabled":["dsh-dream-skin"], ...}          ← 被静默禁用了

Get-Content "$env:USERPROFILE\.dsh\profiles\web\.dsh-market\log.ndjson" -Tail 8
# {"event":"toggle","detail":"dsh-dream-skin -> off: fiber=false"}
# {"event":"boot","detail":"plugin kept off: dsh-dream-skin"}
```

## 三类根因

### ① 被 dshmarket 静默禁用（皮肤消失的真因）

`package.json` 和组合配置里插件都在、`--dump-config` 也能看到它的 bundle 层，
但 **dshmarket 自己的 `state.json` 有一份 disabled 列表**，启动时按它把行关掉。

**修法**：从 `disabled` 数组移除该插件 → 重启。留备份 `state.json.bak`。

### ② 两个插件抢同一个 UI 面（皮肤与壁纸）

dream-skin **自带壁纸功能**：配了壁纸（`dsh-dream-skin:wallpaper-kind`）后它会自己插一个
`position:fixed;inset:0;z-index:-1` 的背景层，还对主题 token 做"水洗调色"（`shadeTokens2`）——
与 wallpaper-engine 的 behind-body 层争同一个面，配色也被洗掉。

**修法（按分工：壁纸归壁纸插件、配色归皮肤）**：清空 dream-skin 的全部壁纸状态键。
在其持久化文件 `~/.dsh/dream-skin.json` 里把 10 个 `dsh-dream-skin:wallpaper*` 键**显式写成 `null`**：

- 客户端 boot 时会从宿主 **adopt** 这些键（含 `null` → 同时删掉浏览器 localStorage 里的残留）
- `null` 墓碑还能阻止**工厂默认壁纸**在下次启动时"复活"

### ③ 缺外部二进制导致静默降级（壁纸不显示）

wallpaper-engine 把 scene（动态场景）壁纸的静态帧与动画 MP4 **全部依赖 ffmpeg 提取**；
宿主会懒下载 ffmpeg-static 到 `~/.dsh-wallpaper-engine/ffmpeg/ffmpeg.exe`，**这次下载失败（目录为空）**，
于是：

- `scene-frame` / `scene-video` 端点 404 → 客户端首选渲染路径全断 → 什么都不显示
- 而 `inventory` 列表照常返回（因为不依赖 ffmpeg），所以"能看到列表却看不到画面"

**修法**：手动按插件锁定的版本与哈希补齐：

```powershell
curl.exe -L -o ffmpeg.exe.part `
  https://registry.npmmirror.com/-/binary/ffmpeg-static/b6.0/ffmpeg-win32-x64
Get-FileHash ffmpeg.exe.part -Algorithm SHA256   # 需等于插件内 FFMPEG_STATIC_SHA256['ffmpeg-win32-x64']
```

实测放入后**无需重启**（插件每次请求都重新检查该文件），`scene-video` 立刻 200（42.9MB MP4）。

## 坑

1. **"装了"≠"生效"**：只看 `package.json` / `--dump-config` 会误判——dshmarket 另有独立禁用状态。
2. **插件包名可能与文档不一致**：文档写 `dsh-wallpaper-engine`，实际装的是 `dsh-plugin-wallpaper-engine`，
   按文档名字找不到时会以为没装。以 `node_modules` 实际目录名为准。
3. **404 与 405 的区分**：路由没注册时，POST 得到的是静态兜底的 405（**空 body**），
   而不是框架的 404——只看状态码会误以为是"方法不对"。
4. **依赖外部二进制的能力会静默降级**：没有 ffmpeg 时插件不报错，只是"渲染不出画面"；
   排查要先看前置依赖（这里是 ffmpeg 目录为空）。
5. **宿主状态文件是权威**：`~/.dsh/dream-skin.json` 里的键（含 `null`）在 boot 时覆盖浏览器 localStorage，
   跨会话/跨端口持久化都靠它——清状态要清这里，只清浏览器缓存无效。

## 验证

```powershell
# dream-skin 恢复：路由返回 200 + 壁纸键全 null
POST /dream-skin/api {"method":"get"}  → 200 {"ok":true,"value":{...,"dsh-dream-skin:wallpaper":null,...}}

# market 日志不再出现 kept off
Select-String log.ndjson -Pattern 'kept off'
```

最终权衡：wallpaper-engine 的 scene 依赖链较重，**卸载该插件改用静态壁纸**，
skin（8 套配色）保留——卸载后两条宿主路由一并消失，皮肤与主题不受影响。

## 教训

1. **排查顺序从"最外层"往里**：路由通不通 → 隔离实例对照 → 模块隔离测试 → 管理器的状态文件。
   这次三步都"正常"，第四步才挖到真凶。
2. **多来源配置要认清谁有最终话语权**：`package.json`（装没装）、组合配置（在不在树里）、
   market state（启没启用）、插件自己的状态文件（功能参数）——四层各有分工。
3. **UI 类插件冲突的本质是"抢同一个渲染面"**：两个插件都画背景层/都改主题 token 时，
   必须明确分工（谁管壁纸、谁管配色），否则表现为随机的"其中一个失效"。
4. **重依赖的插件要评估值不值**：scene 壁纸要 ffmpeg 抽帧转码，链路长、失败静默；
   静态壁纸零依赖——按需求选简单的那个。
