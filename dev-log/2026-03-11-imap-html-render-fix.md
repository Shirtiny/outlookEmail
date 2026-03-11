# 2026-03-11 IMAP 邮件 HTML 渲染修复

## 问题
收到部分邮件时，服务端在 IMAP 路径下优先提取 `text/plain`，导致邮件详情接口返回的正文已经不是 HTML。
这会让前端即使开启“信任此邮件”，也无法显示 HTML 邮件中的按钮、样式和链接布局。

典型现象：
- 邀请邮件中的 `Join workspace` 只显示为纯文本
- 原始 HTML 中存在的 `<a href="...">Join workspace</a>` 按钮无法渲染

## 原因定位
关键逻辑位于：
- `web_outlook_app.py:1135` `get_email_body`
- `web_outlook_app.py:1555` `get_email_detail_imap`

旧逻辑：
- multipart 邮件优先取 `text/plain`
- 取到 plain 后直接结束
- 只有没有 plain 时才退回 `text/html`

因此 HTML part 在服务端被提前丢失。

## 修复内容
### 1. IMAP 正文提取改为优先 HTML
修改 `get_email_body`：
- 同时扫描 `text/html` 和 `text/plain`
- 优先返回 HTML
- 无 HTML 时再回退到纯文本
- 返回值由单个 `body` 改为 `(body, body_type)`

### 2. IMAP 详情接口增加 `body_type`
修改 `get_email_detail_imap` 返回结构：
- `body`
- `body_type`（`html` / `text`）

这样前端现有 `renderEmailDetail` 逻辑可直接识别 HTML 邮件并进入 iframe 渲染链路。

### 3. IMAP 列表预览适配新签名
列表预览仍只使用正文字符串，不改变现有交互。

## 影响范围
仅影响 IMAP 邮件正文提取与详情返回：
- 不改动 Graph API 详情逻辑
- 不改动前端 trust mode 逻辑
- 不改动邮件列表、删除、分页等其他功能

## 结果
修复后，IMAP 路径下的 HTML 邮件可正常返回 HTML 正文。
类似 ChatGPT Business 邀请邮件中的 `Join workspace` 链接按钮可以被前端正常渲染显示。

## 相关文件
- `web_outlook_app.py`
