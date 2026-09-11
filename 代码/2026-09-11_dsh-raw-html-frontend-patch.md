# dsh-raw-html 前端渲染补丁在新核心失效：从源码显示到整条消息不可见

> 日期：2026-09-11 · 环境：Arch Linux (x86_64) · 版本：dsh 0.1.5-rc.2 · 结论：卸载

## 现象

核心从 0.1.1-rc.2 升到 0.1.5-rc.2 后，消息里的裸 HTML 卡片**全部显示为源码**。
按新版压缩产物适配补丁后，症状变成：卡片位置只剩**一条线**（容器在、内容空）；
再修一版后，**整条消息（含正文与思考）都不可见**。

## 背景：这个插件的渲染能力是「打进 dist 的补丁」

`dsh-raw-html` 的架构分两半：

- `patch/patch-frontend.cjs`（含 `install-v6.cjs` + `v6-inject.js`）——**改 `@deepseek-ai/dsh-web-frontend` 的 dist bundle**，
  把 markdown 渲染的 `case"html"` 分支改成调用 `window.__vcpStable.render()`
- 插件本体的 `lib/client.js`——只管「`</>`」开关按钮与设置面板

所以**核心升级 = dist 被整体替换 = 渲染补丁消失**，而补丁不在包管理里，没有自动重打机制。

## 步骤

### 1. 定位失配点：新版压缩产物改名

| 用途 | 旧（0.1.1-rc.x） | 新（0.1.5-rc.2） |
|---|---|---|
| markdown 渲染 switch | `pu()`，参数 `(n,r,i)` | `B3(t,r,i)`，token 变量 `t`、streaming 在第三参数 `i` 上 |
| DOM → React 转换器 | `vc(n,r)` / `Xu(n,i)` | `c8(t,r)`（属性循环变量 `c`/`i`/`s`） |
| style 字符串解析 | `hp` / `jd` / `Sd` | `wg` |
| React 工厂 | `f` | `d` |

补丁脚本报「既非原始态也非已补丁增强态，无法识别」——锚点按压缩后的字面量匹配，变量名一改就全失配。

### 2. 给补丁脚本加「0.1.5 锚点组」

在 `install-v6.cjs` 里加一代探测与替换目标：

- `case"html":return t.value;` → 注入 `window.__vcpStable.render(t.value, i.streaming)`
- `case"code":return Lg(t,r,i);` → 围栏兜底（```html 包卡片时也渲染）
- `for(const c of i.attributes)...wg(c.value)...` → v6 增强属性循环（含 `script/iframe` 过滤、`onclick=input()` 桥、URL 白名单）
- 注入点 `}function c8(t,r){` 之前插入 v6 模块，并把模块内的 React 工厂 `f` 替换成 `d`

### 3.「一条线」的根因：别名 fallback 链少了一代

`v6-inject.js` 用运行时探测兼容多代压缩产物：

```js
function __vcpVc() {
  var fn = typeof vc === 'function' ? vc : (typeof Xu === 'function' ? Xu : null)
  return fn && fn.apply(null, arguments)
}
```

新版是 `c8`，**不在链里** → 返回 `null` → `parseFrag` 里每个 DOM 节点都被转成 `null` →
卡片内容全空，只剩容器边框 = 一条线。

修法：`__vcpVc` 补 `c8`、`__vcpHp` 补 `wg`（style 解析），并让 `update-v6-inject.cjs` 也做 `f → d` 适配。

### 4. 本地复现不出来：真实 React 挂载测试全绿

用 jsdom + React 18.3.1 完整模拟（`renderToString` 验元素树 + `createRoot` 验 ref 回调）：

- SSR 输出 2447 字节、结构完整（`#vcp-root` 正确作用域化为 `#vcp-msg-1`）
- `createRoot` 挂载后 `innerHTML` 2452 字节、内容与样式齐全、**零 console.error**

即**渲染管线本身正常**，崩溃点只在浏览器真实环境（皮肤插件、消息容器结构、模块加载时序）。

### 5. 加错误上报定位——浏览器端零上报

在 dist 顶部注入上报器（`window.onerror` + `unhandledrejection` + hook `console.error`），
双通道发送（`navigator.sendBeacon` → `/report`，`Image.src` → `/beacon`），
本地起收集服务（`127.0.0.1:3099`）并带**自动降级**（收到渲染错误就把 `case"html"` 改回源码模式）。

结果：刷新后**一条上报都没收到**——怀疑浏览器仍在用缓存里的旧 bundle。

### 6. 绕缓存与最终处置

- 给 `index.html` 的 script 引用加版本串、并把 bundle **改文件名**（`-p2` → `-p3`）强制重新下载
- 仍未能确认渲染恢复；用户决定不再维护该插件
- 最终：卸载插件 + 把 dist bundle 从**原始备份**恢复（补丁特征清零）+ 删除所有临时文件名与 `.bak`

## 坑

1. **dist 补丁没有任何自动重打机制**：核心升级覆盖文件后，补丁静默消失，症状要等用户发现。
2. **锚点匹配压缩产物 = 每次核心升级都可能全失配**：变量名（`vc`→`c8`、`f`→`d`）是最脆弱的一环。
3. **浏览器缓存会让「改了没生效」误判**：文件名不变时 F5 可能直接命中缓存；
   判断依据不是「刷新了」，而是**产物里有没有探针上报**。正确做法：改文件名或带版本串。
4. **本地全绿 ≠ 线上正常**：jsdom + React 能验证元素树合法性，但验证不了皮肤插件、
   消息容器结构、模块加载顺序这些真实环境因素。
5. **容器还在但内容空**：这类「一条线」症状优先查「节点转换函数返回了 null」，
   而不是查 CSS。

## 验证

```bash
# 卸载后
dsh --profile web --dump-config      # EXIT 0，stderr 0 字节
```

- `package.json` 的 dependencies 与 bundles 中均无 `dsh-raw-html`
- dist bundle 恢复原始态：`__vcpStable` / `__DSH_V6_INJECT` / 诊断上报器**特征计数全为 0**，`node --check` 通过
- `index.html` 引用恢复为原始文件名；临时文件（`-p2` / `-p3` / `.bak-*`）已删
- web 正常启动，历史消息以源码文本形式完整可见

## 教训

1. **给第三方 dist 打补丁的能力天生脆弱**：上游一升级就全丢，且补丁维护成本随上游压缩产物变化而上升。
   评估这类方案时要把「每次升级都要重新适配」算进成本。
2. **排查「改了没效果」先证伪加载**：加一个「代码执行即上报」的探针，
   比反复刷新页面猜缓存可靠得多。
3. **保留原始备份是底线**：本次能干净回滚，靠的是打补丁前对 dist 的完整备份。
4. **维护成本 > 收益时，卸载是合理选择**：这个插件的核心价值（卡片渲染）依赖最脆弱的一层，
   卸载后消息以源码形式显示，信息不丢，只是不美观。
