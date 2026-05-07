---
name: web-automation
description: 浏览器与网站自动化智能路由。专注 Chrome 浏览器操作、动态页面读取、截图、页面交互、网络抓包、控制台分析与下载处理。已移除 OpenCLI；所有浏览器自动化统一走 chrome-mcp-server。
---

# Web Automation — Chrome 优先

这个 skill 只做浏览器与站点自动化，不替代通用搜索。

## 工具边界

- **开源库 / 框架 / SDK / API 文档** → 先用 `context7`
- **公开信息发现 / 通用搜索** → 首选 `grok-search`，且只用 `web_search`
- **已知 URL，且页面偏静态正文、文档、博客、PDF** → 用 `exa-pool` 的 `exa_get_contents`
- **需要登录态、动态渲染、页面交互、上传、截图、抓包，或静态提取本来就不适合** → 进入 `web-automation`

> `grok-search` 的 `web_fetch` / `web_map` 在这里禁用。已知 URL 内容提取优先 `exa_get_contents`；动态页面才进 Chrome。

## 核心原则

1. **能不进浏览器就不进**
2. **进了浏览器先走最省调用路径**
3. **只为完成任务做必要交互**
4. **截图一律存文件，不塞 base64**
5. **本 skill 不再依赖 OpenCLI**

## 何时升级到浏览器

出现以下任一情况，别继续停留在搜索层：

- 已知 URL 在静态层拿不到有效内容
- 页面是数据面板、复杂列表、商品页、后台、创作者平台
- 目标站点强依赖登录态、前端渲染或用户交互
- 任务要求上传文件、提交表单、点击展开、切换分页
- 需要截图、录屏、控制台、网络请求或页面内 JavaScript
- 目标是强动态站点，且结果必须来自当前页面状态

## 浏览器内路由

### 1. 只要正文或可见文本

优先：

```text
chrome_get_web_content(textContent=true)
```

适合：

- 文章正文
- 文档页面
- 已渲染完成的可见文本

### 2. 需要结构化数据提取

优先：

```text
chrome_javascript(code="return ...")
```

适合：

- 列表卡片
- 表格
- 页面内 JS 变量
- 需要一次性提取数组/对象

比 `read_page → 手工解析` 更省。

### 3. 需要找可点击元素

优先：

```text
chrome_read_page(filter="interactive")
```

然后按情况用：

- `chrome_click_element`
- `chrome_fill_or_select`
- `chrome_keyboard`
- `chrome_computer`

### 4. 需要复杂交互

用 `chrome_computer`：

- 拖拽
- 滚动
- hover
- GIF 录制配套操作
- 特殊布局或 ref 不稳定页面

### 5. 需要抓包 / 控制台 / 下载

- 网络抓包：`chrome_network_capture`
- 控制台：`chrome_console`
- 下载等待：`chrome_handle_download`

## 决策流程图

```text
用户任务
  ├─ 是开源库/框架/SDK/API 文档查询？
  │   └─ → context7
  ├─ 只是公开信息发现 / 通用搜索？
  │   └─ → grok-search web_search
  ├─ 已知 URL，且适合静态正文提取？
  │   └─ → exa-pool exa_get_contents
  ├─ 需要登录态、动态页面、交互、截图或抓包？
  │   └─ 是 ↓
  ├─ 只需要页面文本？
  │   └─ → chrome_get_web_content
  ├─ 需要结构化提取？
  │   └─ → chrome_javascript
  ├─ 需要点击/填写/滚动？
  │   └─ → read_page(filter="interactive") → click/fill/computer
  ├─ 需要截图或录屏？
  │   └─ → screenshot / gif_recorder
  ├─ 需要网络抓包/API 分析？
  │   └─ → network_capture
  └─ 不确定？
      └─ → 先 get_web_content；不够再升级
```

## 截图规则

`chrome_screenshot` / `chrome_computer(action="screenshot")` 必须用 `savePng: true`。**禁止 `storeBase64`**。

原因：

- base64 文本体积大，直接污染上下文
- 几张图就可能把上下文拖爆

实践：

- 局部细节优先用 `selector`
- 截图后先跑：

```bash
OPT=$(ls ~/.claude/skills/jarvis/scripts/optimize-image.sh ~/.codex/skills/jarvis/scripts/optimize-image.sh 2>/dev/null | head -1) && bash "$OPT" <路径>
```

然后再读图

## chrome-mcp-server 高效模式

### 减少调用次数

1. 文本优先 `chrome_get_web_content`
2. 结构化提取优先 `chrome_javascript`
3. 只要可交互元素时，`chrome_read_page(filter="interactive")`
4. 多字段填写优先 `chrome_computer(action="fill_form")` 或批量填充
5. 只有 ref 不稳定或布局怪时，才上坐标点击

### 连接异常先断开重连

`chrome-mcp-server` 出现 `server unavailable`、`browser disconnected`、`target closed` 这类 transport 错误时：

1. 先别误判成站点坏了
2. 不要原地重试同一复杂调用超过 2 次
3. 先做一次断开重连
4. 重连后先用 `get_windows_and_tabs`、`chrome_get_web_content(textContent=true)` 或 `chrome_read_page(filter="interactive")` 做最小探针
5. 探针通过再继续；仍失败才降级到静态提取 / 搜索，必须依赖登录态或动态交互则明确报阻塞

### 常用操作速查

```text
# 页面文本
chrome_get_web_content(textContent=true)

# 可交互元素
chrome_read_page(filter="interactive")

# 执行 JS 提取
chrome_javascript(code="return ...")

# 截图
chrome_screenshot(savePng=true)

# 录制 GIF
chrome_gif_recorder(action="auto_start") → 操作 → chrome_gif_recorder(action="stop")

# 网络抓包
chrome_network_capture(action="start") → 操作 → chrome_network_capture(action="stop")

# 控制台
chrome_console(mode="snapshot")
```

## 并行调研提示

只有目标彼此独立时才并行，例如同时看多个官网、多个后台页面、多个站点登录态结果。

给子任务时描述目标，不要把手段写死：

- 好：`调研这 5 个官网的定价、核心卖点和截图，给我摘要`
- 差：`先搜索，再用某工具抓，再截图`

目标清楚，执行者才知道先走搜索、静态提取，还是直接进浏览器。

## Site Patterns

站点经验放在 [`references/site-patterns/`](./references/site-patterns/)。

规则：

1. 先查是否已有目标域名文件
2. 命中经验则优先复用
3. 经验失效时回退通用模式
4. 确认新规律后再更新
5. 只记录验证过的事实

模板见 [`references/site-patterns/TEMPLATE.md`](./references/site-patterns/TEMPLATE.md)。
