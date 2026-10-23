# 各平台爬取工具速查

按平台选工具。所有工具由 [agent-reach](https://github.com/Panniantong/Agent-Reach) 路由（抖音除外，走 douyin-mcp）。安装后可先运行 `agent-reach doctor --json` 检查各平台后端状态。

## 抖音（douyin-mcp）

```
# 获取视频信息
mcp__douyin-mcp__parse_douyin_video_info(share_link="<URL>")
# 提取字幕（语音转文字）
mcp__douyin-mcp__extract_douyin_text(share_link="<URL>")
```

要点：抖音字幕不在 HTML 里，普通网页抓取只能拿到标题和标签，必须走 ASR 转录。douyin-mcp 的 `mcp` 依赖必须锁定 1.20.0（2.0 起删除了 FastMCP 模块），并用本地 venv 直连，不要通过 `uvx` 启动。需配置 `DASHSCOPE_API_KEY` 或 `API_KEY`。

## B站

先从 URL 提取 BV 号（如 `BV1eZgt6YEP5`）。

```bash
# 视频信息
bili video <BV号>
# 字幕：优先 opencli（覆盖率明显高于 bili-cli，连 AI 生成字幕也能拿到）
opencli bilibili subtitle <BV号>
```

要点：
- 不要尝试 `bili audio`（pipx 隔离环境装 PyAV 很麻烦）。
- 即使 bili-cli 报告无 CC 字幕，也要再试 opencli —— 两者字幕数据源不同。
- opencli 报 `BROWSER_CONNECT`：需要 Chrome 已打开、已登录 B站、OpenCLI 扩展已启用。
- `bili video` 失败时的备用：`opencli bilibili video <BV号> -f yaml`。

## 小红书

```bash
opencli xiaohongshu note "<URL>" -f yaml
```

要点：需要 Chrome 已登录小红书且 OpenCLI 扩展已连接。

## YouTube

```bash
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --skip-download -o "/tmp/%(id)s" "<URL>"
cat /tmp/<VIDEO_ID>.*.vtt
```

## X / Twitter

```bash
twitter tweet "<URL>"
```

## 通用网页

```bash
curl -s "https://r.jina.ai/<URL>"
```

要点：Jina Reader 返回正文 Markdown，适合作为任何平台的兜底方案（视频类内容拿不到字幕，只能取页面文本）。
