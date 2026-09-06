# B 站扫码登录在 Linux 上失效

> 日期：2026-09-06 · 项目：VideoToNo v1.3.5 · 上游文件：backend/bili_login.py

## 现象

点「扫码登录」立即失败，返回：`未找到 Edge 或 Chrome，请手动填写凭据`。
机器上明明装着 Chrome（`/usr/bin/google-chrome-stable`）。

## 根因

`bili_login.py` 的 `BROWSER_PATHS` 只声明了 Windows（固定安装路径）和
macOS（/Applications）的浏览器路径，**完全没有 Linux 分支**。
`find_browser()` 只做 `os.path.isfile()` 检查，在 Linux 上恒返回 None。

## 修复

1. `BROWSER_PATHS` 补 Linux 常见安装路径（`/usr/bin`、`/snap`、flatpak 导出目录）
2. `find_browser()` 增加 PATH 兜底：`shutil.which()` 按可执行名查找
   （google-chrome / chromium / microsoft-edge 等），覆盖自定义安装位置

## 验证

- `find_browser()` → `/usr/bin/google-chrome-stable` ✓
- `POST /api/bili-login/start` → `{"ok": true, "browser": "google-chrome-stable"}` ✓
- Chrome 以独立 profile + CDP 调试端口（9333）拉起 B 站登录页，状态 waiting ✓

## 教训

跨平台工具最容易漏的就是 Linux：写死路径列表时，永远记得加一层
`shutil.which()` 兜底——flatpak / snap / 自定义编译安装的位置根本枚举不完。
