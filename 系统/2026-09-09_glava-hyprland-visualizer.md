# GLava 圆形频谱 · Hyprland 配置与排障全程

日期：2026-09-09 · 环境：Arch + Hyprland 0.56.2（Lua 配置）+ GLava 1.6.3 + PipeWire(PulseAudio) + 网易云

## 背景与目标

在屏幕中央放一个精致的圆形音乐频谱（GLava radial 模块）：细密发光、白蓝渐变、透明背景、垫底在桌面层（应用之下、壁纸之上）、待机不单调。不装"一堆"软件，只动必要的配置。

## 最终成果（结论先行）

- **形状**：圆心 "Iris" 字样 → 内环(30px) → 主环(54px)+光晕 → 待机刻度(60px) → 圆形波形带（外伸最长 ~250px，约 1cm 律动）
- **垫底层级**：通过 hyprwinwrap 插件把 GLava 窗口钉在壁纸之上、所有应用之下（空桌面可见，窗口盖住它）
- **音频**：跟随默认 sink（耳机/扬声器切换后需重启 glava 重新解析）
- **启动**：`glava --desktop`

## 涉及的文件（都在用户目录，未动 /etc）

| 文件 | 作用 |
|---|---|
| `~/.config/glava/rc.glsl` | 窗口属性：floating/decorated/geometry/clickthrough/setxwintype 等 |
| `~/.config/glava/radial.glsl` | 全部视觉宏（半径/条数/幅度/发光/待机装饰） |
| `~/.config/glava/radial/1.frag` | 自定义波形 shader（Catmull-Rom 波形 + 发光 + Iris 字样位图） |
| `~/.config/glava/smooth_parameters.glsl` | 平滑/平均帧参数 |
| `~/.config/hypr/custom/rules.lua` | Hyprland 窗口规则 + hyprwinwrap 垫底配置 |
| `~/.local/lib/hyprland/plugins/libhyprwinwrap.so` | hyprwinwrap 插件（手动编译，源码 `~/src-hyprwinwrap/`） |

## 核心配置

### Hyprland 窗口规则（rules.lua）

```lua
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, float = true })
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, pin = true })       -- 跨工作区可见
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, border_size = 0 })  -- 去黑边
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, no_shadow = true })
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, no_anim = true })
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, no_focus = true })
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, center = true })
hl.window_rule({ match = { class = "^[Gg][Ll]ava$" }, immediate = true })

-- hyprwinwrap：垫底到壁纸之上、应用之下（520×520 居中）
hl.plugin.load("/home/yuan/.local/lib/hyprland/plugins/libhyprwinwrap.so")
if hl.plugin.hyprwinwrap ~= nil then
    hl.plugin.hyprwinwrap.window({
        class = "GLava", title = "GLava", layer = 0,
        pos_x = 39, pos_y = 32, size_x = 22, size_y = 36
    })
end
```

> ⚠️ XWayland 下 WM_CLASS 是大写 `GLava`，匹配必须写 `^[Gg][Ll]ava$`。

### radial.glsl 关键宏

```
C_RADIUS 54        -- 主环半径（小）
NBARS 240          -- 波形采样密度
AMPLIFY 195        -- 波形外伸幅度（≈1cm 律动）
setgravitystep 2.8 -- 回落速度（越小越优雅）
setsmoothfactor 0.05
OUTLINE (#d8e9ff * 0.85)  -- 内圈亮蓝白
RING_GLOW 4.0 / IDLE_BAR 7.0 / IDLE_ALPHA 0.30 / CENTER_DOT_R 11.0 / INNER_RING_R 30.0
```

## 排障历程（现象 → 根因 → 修复）

### 1. 默认红色粗条，不想要
- **修复**：重写 `radial/1.frag`——把频谱值用 **Catmull-Rom 四点插值**连成连续圆形波形带（细密、丝滑），白→浅蓝渐变，上缘发光。

### 2. `no_border` 字段报错（Hyprland 配置加载失败 → 安全模式）
- **根因**：Hyprland 0.56.2 的 `hl.window_rule` **没有 `no_border` 字段**（那是 workspace rule 的字段）。
- **修复**：拉取 v0.56.2 源码 `LuaBindingsInternal.hpp` 核对完整字段表 → 用 `border_size = 0`。
- **教训**：改 Hyprland 配置前先查目标版本的合法字段；改完立刻 `hyprctl reload` 验证。

### 3. GLava 浮在应用上面，想要"桌面层"
- **根因**：Hyprland 的 floating 窗口层**恒高于 tiled 层**；X11 的 `_NET_WM_WINDOW_TYPE_DESKTOP` 只影响焦点不影响 z 序（查源码确认）。
- **修复**：装 **hyprwinwrap** 插件（`gen3vra/hyprwinwrap`，需 Hypr 0.54+，兼容 0.56.2）。hyprpm 在本环境因 pkexec 无 polkit agent 失败 → 改**手动编译**：clone → cmake → `libhyprwinwrap.so` 放用户目录 → `hl.plugin.load(...)` 加载，完全不需要 root。

### 4. 波形"冻结"假象（画面静止只有圈）
- **根因**：GLava 的音频源 `setsource "auto"` **只在启动时解析一次默认 sink**。当蓝牙耳机/扬声器切换后，GLava 仍挂在旧的（或麦克风）源上采静音。
- **修复**：切换音频设备后 `pkill glava; glava --desktop` 重启即可重新跟随。现象与"渲染冻结"几乎一样，容易误判（详见第 7 条）。

### 5. 双实例重叠（自己手动又启了一个）
- **根因**：手动 `glava` 时旧实例还在 → 两个窗口完全重叠，旧实例被判定遮挡冻结（FPS 1.0）。
- **修复**：`pkill -x glava` 后只保留一个实例。

### 6. 黑边 / 黑色细环
- 黑边 = Hyprland 窗口 border + shadow → `border_size = 0` + `no_shadow`（仅 GLava 窗口）。
- 内圈细环在暗壁纸上发黑 → 提亮 OUTLINE 到 0.85 + 加 RING_GLOW 光晕。

### 7. 待机单调 → 逐层加装饰
- 中心光点 → 内装饰环 → 待机刻度 → 最终换成 **"Iris" 字样**（Google Sans Flex 渲染成 48×29 位图内嵌 shader，白字+深蓝紫阴影，亮壁纸也清晰；低音时微呼吸）。
- **踩坑**：`-resize 44x16!` 强制非等比缩放会把文字压扁变形 → 必须等比缩放。

### 8. `time` uniform 绑定失败（想做呼吸动画）
- **根因**：GLava 1.6.3 源码 bug——`render.c` 里 `"time"` 条目注册时 `.src_type = SRC_SCREEN`（应为 SRC_TIME），绑定必失败。
- **结论**：GLava 1.6.3 无法用 time uniform，呼吸动画放弃（用音频 bass 值驱动替代）。

### 9. hyprwinwrap 场景"GLava 不渲染"误判
- **真相**：hyprwinwrap 把窗口标为 `visible: False`，但 **X11 可见性实际正常**（`XGetWindowAttributes` 返回 IsViewable），GLava 一直在渲染；"静止"其实是第 4 条的音频源问题。
- **验证方法**：LD_PRELOAD 拦截 X 调用打日志（`/tmp/glavasrc` 看 `should_render` 逻辑）——`glx_wcb.c` 的 `should_render` 只有 `map_state == IsViewable` 一个硬条件。

### 10. 配置损坏事故（安全模式）
- **根因**：调试时用 sed/python 注释脚本只注释了 `hl.plugin.hyprwinwrap.window({` 一行，表内字段变成孤立顶层语句 → Lua 语法错误 → Hyprland 回退无配置状态。
- **修复**：从备份恢复 + 补回 `pin`，`hyprctl reload` 验证 ok。
- **教训**：改 Lua 配置文件**整文件重写**（write 工具），别用注释脚本；改前备份；改后立刻 reload 验证。

## 调参速查（都改 radial.glsl，改完重启 glava）

| 想调什么 | 改哪个 |
|---|---|
| 圈大小 | C_RADIUS |
| 波形长短 | AMPLIFY |
| 律动快慢/优雅 | setgravitystep（回落速度）/ setsmoothfactor |
| 待机刻度 | IDLE_BAR / IDLE_ALPHA |
| 内环装饰 | INNER_RING_R / RING_GLOW |
| 环的颜色 | OUTLINE |
| Iris 字样 | 重新渲染位图（magick + 等比缩放）嵌入 1.frag |

## 教训清单（下次避免）

1. **Hyprland 0.56 用 Lua 配置**：改前查版本源码的合法字段；整文件写、改前备份、改后 `hyprctl reload` 验证。
2. **GLava 音频源一次性解析**：切音频设备后必须重启 glava。
3. **GLava 遮挡判定**：靠 X11 VisibilityNotify + map_state；背景型窗口（hyprwinwrap）会让它停在最后一帧的"假象"——先查音频源再查渲染。
4. **XWayland 窗口 class 大写**：`GLava` 不是 `glava`。
5. **文字位图必须等比缩放**，否则变形。

## 回滚

```bash
pkill glava
# GLava 配置备份在：~/.config/glava/backup-20260909_141829/
# rules.lua 备份：~/.config/hypr/custom/rules.lua.bak-1750
hyprctl reload
# 去掉 hyprwinwrap：注释 rules.lua 末尾 hyprwinwrap 段，reload
# 插件文件：~/.local/lib/hyprland/plugins/libhyprwinwrap.so（删掉即失效，无需卸载）
```
