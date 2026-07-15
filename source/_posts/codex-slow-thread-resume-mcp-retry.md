---
title: "Codex 旧会话打开慢：MCP 重试"
date: 2026-07-15 11:20:07
tags:
  - Codex
  - MCP
  - Shadowrocket
  - 性能排查
categories:
  - 问题排查
---

> **运行环境**
> - 系统：macOS 26.5.1（arm64）
> - 工具：Codex Desktop 26.707.72221
> - 网络：Shadowrocket，系统 HTTP/HTTPS 代理为 `127.0.0.1:1082`
> - 场景：关闭多个不常用插件后，打开 Codex 旧会话仍需等待几十秒

## 问题现象

Codex Desktop 打开一个之前用于修改 Crux UI 的旧会话时，界面长时间停留在恢复状态。此前已经关闭了多个不常用插件，但这次恢复仍然用了 **43.412 秒**。

同一时段还有一次旧会话恢复耗时 **75.632 秒**。因此，问题显然不只是“插件数量多”。

## 排查过程

### 1. 先区分读取、恢复和渲染

桌面日志位于：

```text
~/Library/Logs/com.openai.codex/YYYY/MM/DD/
```

根据同一次打开操作中的耗时记录：

```text
thread/read        3 ms
thread/resume      43412 ms
thread/turns/list  43 ms / 65 ms / 51 ms
```

`thread/read` 和恢复后的分页查询都在毫秒级，说明 SQLite 查询和前端渲染不是主要瓶颈，等待集中在后端的 `thread/resume`。

### 2. 检查旧会话文件解析

后端日志显示，Codex 在恢复开始的同一秒就完成了会话文件解析：

```text
Resumed rollout with 1795 items ... parse errors: 0
```

虽然这个会话历史较长，但读取 1795 条记录并没有消耗几十秒。历史体积可能增加基础成本，却无法解释本次 43 秒的等待。

### 3. 沿 `thread/resume` 查看初始化过程

Codex 的结构化后端日志保存在：

```text
~/.codex/logs_2.sqlite
```

可以先搜索恢复期间的错误：

```bash
sqlite3 ~/.codex/logs_2.sqlite \
  "SELECT datetime(ts, 'unixepoch'), level, target, feedback_log_body
   FROM logs
   WHERE feedback_log_body LIKE '%error decoding response body%'
   ORDER BY ts DESC
   LIMIT 20;"
```

这次恢复主要出现了两段额外等待：

1. 模型列表缓存过期，远程刷新约耗时 5 秒，随后记录了：

   ```text
   failed to refresh available models: timeout waiting for child process to exit
   ```

2. 初始化 `openaiDeveloperDocs` MCP 时，访问 `developers.openai.com` 连续失败：

   ```text
   resource metadata probe failed: error decoding response body
   discovery request failed: error decoding response body
   ```

这些请求大约每 5 秒失败一次，持续重试到 `02:51:30`，恰好与 `thread/resume` 完成时间重合。

### 4. 检查代理和 Codex 配置

系统代理可以通过以下命令查看：

```bash
scutil --proxy
```

本机结果为：

```text
HTTPProxy  : 127.0.0.1
HTTPPort   : 1082
HTTPSProxy : 127.0.0.1
HTTPSPort  : 1082
```

对应的监听进程来自 Shadowrocket 的 `MacPacketTunnel`。日志中 `developers.openai.com` 还被解析到了 `198.18.0.28`，这是代理软件常用的 Fake IP 地址范围。

继续检查 `~/.codex/config.toml`：

```toml
[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
```

这里有一个容易混淆的点：`openaiDeveloperDocs` 是**全局 MCP Server**，不是普通插件。关闭 Documents、PDF、Slides 等插件并不会关闭它。

## 原因

旧会话恢复时，Codex 不只是读取历史记录，还会重新初始化当前会话需要的模型信息和 MCP Server。

本次真正的主要耗时来自 `openaiDeveloperDocs` MCP：它通过当前代理访问 `developers.openai.com` 时拿到了无法解析的响应，OAuth/资源元数据发现流程按约 5 秒间隔反复重试，最终把一次本应很快的恢复拖到了 43 秒。

Browser 后端在约 5 毫秒内就已就绪，因此 Browser/Chrome 插件不是这次等待的主因。运行较久后残留的多个 `node_repl` 子进程会让状态更复杂，但也不是这 43 秒的直接来源。

## 解决方案

### 方案一：不常用官方文档检索时，关闭该 MCP

`openaiDeveloperDocs` 主要用于让 Codex 搜索和读取最新的 OpenAI 官方开发文档。普通代码编辑、Git、终端和项目分析不依赖它。

在 `~/.codex/config.toml` 中增加：

```toml
[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
enabled = false
```

修改后需要**彻底退出并重新打开 Codex Desktop**。仅关闭窗口不一定会结束后台的 app-server 和 `node_repl` 进程。

需要查询 OpenAI 最新 API 文档时，再临时启用并重启即可。

### 方案二：保留 MCP，修复代理链路

如果经常使用官方文档检索，应在 Shadowrocket 中检查 `developers.openai.com` 的节点、规则和 Fake IP 处理，确保 MCP 端点返回正确的 HTTP/MCP 响应，而不是 HTML 错误页或被截断的内容。

修复后重新打开同一旧会话，并比较以下指标：

```text
thread/read
thread/resume
thread/turns/list
```

只要 `error decoding response body` 不再按 5 秒间隔出现，`thread/resume` 就不应再承担这段重试时间。

## 小结

排查 Codex 旧会话打开慢时，不要只看会话大小或插件数量。先比较 `thread/read`、`thread/resume` 和 `thread/turns/list`：如果只有 `thread/resume` 慢，就继续沿后端日志检查模型刷新、MCP 初始化和网络代理。

这次的关键结论是：**插件关闭已经生效，但一个独立启用的全局 MCP 在代理链路上连续重试，才是旧会话恢复几十秒的主要原因。**
