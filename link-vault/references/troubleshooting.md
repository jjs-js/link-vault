# 排障经验（来自实际部署记录）

## 抖音内容获取

| 问题 | 根因 | 解法 |
|---|---|---|
| 网页抓取只能拿到标题/标签 | 抖音字幕不在 HTML 中 | 必须用 douyin-mcp 做 ASR 语音转文字 |
| douyin-mcp 启动即崩 | `uvx` 拉到的 `mcp` 2.0 删除了 FastMCP 模块 | 本地 venv 锁定 `mcp==1.20.0`，配置直连本地 Python，不用 `uvx` / `uv run` |
| SiliconFlow 返回 HTTP 402 | 免费额度用完 | 充值或换转录通道 |

通用教训：`uvx` / `uv run` 不会复用已有 venv，会从 PyPI 重新拉依赖；依赖出现 breaking change 时会无声地坏掉。直连自己控制好的本地 venv 才可控。

## Fast Note Sync（FNS）部署

| 问题 | 根因 | 解法 |
|---|---|---|
| `bootstrap` 命令不存在 | v3.6.0 已移除该命令 | 发行包自带 `config.yaml`，直接 `fast-note-sync-service.exe run` |
| Android 连不上电脑 IP | 有线网卡与手机热点是两个网段 | 手机改用热点网关 IP（如 `192.168.137.1:9000`） |
| 能 ping 通但连不上 | Windows 防火墙拦截 9000 端口 | 放行规则：`netsh advfirewall firewall add rule name="FNS" dir=in action=allow protocol=TCP localport=9000` |
| Obsidian 插件连不上 | Clash 等代理软件劫持 WebSocket | 关闭代理或为局域网地址设直连 |

## FNS MCP 连接

| 问题 | 根因 | 解法 |
|---|---|---|
| `Scope restricted: Permission denied: /api/mcp` | GUI「Copy API Config」生成的 Token 默认 Scope 只有 `p:rest`，不含 MCP 权限 | 在 Token 管理页手动新建 Token，**Scope 留空**（空 = 全权限 `p:* c:* f:*`） |
| 换了 Token 仍报 Scope Restricted | 换的仍是「Copy API Config」那串，Token 实际没换 | 确认 Token ID 已变化，新建的才是空 Scope |
| MCP SSE 间歇性 `unexpected EOF` | 客户端长连接断开 | 不影响同步：FNS 服务端自动检测文件变更并同步，不依赖 MCP 调用触发 |

## B站字幕

| 问题 | 根因 | 解法 |
|---|---|---|
| `bili video --subtitle` 返回无字幕 | 视频本身无 CC 字幕 | 改用 opencli，它还能拿到 AI 自动生成字幕 |
| opencli 报 `BROWSER_CONNECT` | Chrome 未打开或扩展未连接 | 打开 Chrome、登录 B站、启用 OpenCLI 扩展 |

## 笔记写入

| 问题 | 根因 | 解法 |
|---|---|---|
| obsidian-cli 写长内容报 exit 127 | 正文含 `---`、`$`、引号等，shell 解析异常 | 写笔记统一用 bash heredoc；obsidian-cli 只做 move/search/property 等短操作 |
| 文件工具写 Vault 被拒 | 沙箱只允许写工作区内路径 | 用 bash 写文件 |
| heredoc 里 `$` 被展开 | 分隔符未加单引号 | 必须用 `<< 'ENDOFFILE'` |

## 本地转录（已放弃的路线，仅作记录）

从国内网络直连 HuggingFace 下载 Whisper 模型基本不通（LFS 指针、镜像、huggingface_hub 均失败）。最终方案是云端 ASR（SiliconFlow SenseVoice），本地只做下载和音频提取。
