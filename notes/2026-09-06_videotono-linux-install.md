# VideoToNo Linux 安装记录

> 日期：2026-09-06 · 环境：Arch Linux (x86_64) · 版本：VideoToNo v1.3.5

## 现象

想把 [like-attract/video-to-note](https://github.com/like-attract/video-to-note)
（视频 → 带时间轴 Markdown 笔记的本地服务）在 Arch Linux 上装起来。
官方主推 Windows 便携版 exe，Linux 只能走源码。

## 环境

| 项 | 值 |
|---|---|
| 系统 | Arch Linux (x86_64) |
| Python | 3.11.15（虚拟环境） |
| 系统依赖 | ffmpeg n9.0.1 · yt-dlp 2026.08.19（系统已装） |
| 包管理 | uv 0.12.7 |

## 步骤

```bash
git clone https://github.com/like-attract/video-to-note.git
cd video-to-note
uv venv --python /home/yuan/.local/bin/python3.11 .venv   # 见「坑 1」
uv pip install -r backend/requirements.txt
VIDEOTONOTES_NO_BROWSER=1 .venv/bin/python launcher.py    # 启动
```

健康检查：`GET /api/health` → `{"status":"ok","service":"VideoToNo","version":"1.3.5"}`

## 坑

1. **Python 版本**：项目要求 3.11，系统默认 3.14.7。faster-whisper 依赖的
   CTranslate2 大概率没有 3.14 的 wheel，必须用 3.11 虚拟环境（uv 一条命令搞定）。
2. **浏览器自动打开**：无桌面会话时 launcher 会调 `webbrowser.open`，
   加 `VIDEOTONOTES_NO_BROWSER=1` 可禁用。
3. **系统依赖**：ffmpeg / yt-dlp 是硬依赖，缺了转写和下载全挂。

## 验证

- `/api/health` 返回 ok，三大依赖（yt_dlp / faster_whisper / openai）全部就绪
- 前端页面 HTTP 200
- MCP 端点 `/mcp/sse` HTTP 200（agent 接入可用）

## 结论

Linux 源码安装可行：ffmpeg/yt-dlp 装好 + Python 3.11 venv + pip 装依赖即可。
首次 Whisper 转写会自动从 hf-mirror 下载模型（约 75MB+），缓存于 `workspace/_model_cache/`。
