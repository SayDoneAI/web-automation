# web-automation

浏览器与网站自动化 skill。定位很窄：

- 已知 URL 正文提取失败，或页面本身就是动态站点
- 需要登录态、交互、截图、上传、抓包、控制台、下载
- 需要直接操控 Chrome，而不是停留在搜索层

这个仓库已经**移除 OpenCLI**。浏览器自动化统一走 `chrome-mcp-server`。

## 安装

```bash
ln -sfn ~/Documents/RedCode/web-automation ~/.claude/skills/web-automation
ln -sfn ~/Documents/RedCode/web-automation ~/.codex/skills/web-automation
```

## 依赖

- Chrome
- `chrome-mcp-server`
- 已登录的浏览器 profile（如果任务需要登录态）

## 仓库内容

- `SKILL.md`：skill 主规则
- `evals/evals.json`：路由评测样例
- `references/site-patterns/`：按域名沉淀的站点经验

## 设计原则

- 搜索、文档、静态正文提取先走更轻工具
- 真要进浏览器，再进浏览器
- 进入浏览器后优先最省调用路径：`get_web_content` → `chrome_javascript` → `read_page` → 交互
- 截图禁止 base64 入上下文，统一存文件后读取
- `chrome-mcp-server` 整体失灵时，先断开重连，再用最小探针确认恢复，不要直接把问题归因给目标站点

## 故障处理

`chrome-mcp-server` 出现连接断开、browser disconnected、target closed、server unavailable 这类 transport 层错误时：

1. 先别误判成站点坏了
2. 不要原地重试同一复杂调用超过 2 次
3. 先做一次断开重连
4. 重连后先用 `get_windows_and_tabs` 或轻量页面读取做最小探针
5. 探针通过再继续；仍失败才降级到静态提取或搜索，必须依赖登录态/动态交互则明确报阻塞
