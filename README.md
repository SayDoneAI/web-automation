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
