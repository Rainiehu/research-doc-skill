# 飞书 docx 发布全流程与踩坑

调研文档默认发成飞书 docx。这份文件是把「建文档 → 设标题 → 归档 → 发链接」这条链路的所有坑固化下来的操作手册。**用户不在飞书环境时，整节跳过，改用 Markdown / 本地文件，骨架不变。**

> 依赖 `lark-cli`（lark-channel-bridge 环境会自动注入 bridge-bound 配置）。命令里凡涉及中文，一律先写文件再 `@file` 引用，命令行不出现中文（否则 shell hook 挂起）。

---

## 标准链路（5 步）

### 1. 用 bot 身份建文档
- 默认 `--as bot`（机器人自己），归属稳定、不依赖用户登录态。
- 用 `docs +create`（v2，docxXML 或 Markdown）写入正文。
- ⚠️ **不要指望 docxXML 的 `<title>`**——v2 创建接口不读它，会建成 `Untitled`。

### 2. 显式设标题（必做）
- 建完后**显式调 `update_title`** 设标题，标题加前缀 `[YuCopilot] `。
- 标题写文件再引用：
  ```bash
  # title.json = {"title":"[YuCopilot] <对象> 调研"}   ← 用 create 工具写，命令行不出现中文
  lark-cli api POST /open-apis/docx/v1/documents/<doc_id>/... # 或挂进 wiki 后用 wiki update_title
  ```
- ⚠️ **整篇 overwrite 会把标题重置成 Untitled**——必须 overwrite 后重设标题并核验。
- 核验：`lark-cli api GET /open-apis/docx/v1/documents/<doc_id>` 看 title。

### 3. 归档进知识库（默认动作）
- `docs +create` 默认落个人云盘 = 游离文档、不在知识库。**建完立即 move**：
  ```bash
  lark-cli wiki +move --obj-type docx --obj-token <docToken> \
    --target-space-id 7647550775677226188 --as bot   # 7647... = AI Related
  ```
- move 成功返回 `wiki_token`；用户链接用 `https://my.feishu.cn/wiki/<wiki_token>`。
- 挂进 wiki 后改标题用 wiki 接口：
  ```bash
  lark-cli api POST /open-apis/wiki/v2/spaces/<space_id>/nodes/<wiki_token>/update_title \
    --data @./title.json --as bot
  ```

### 4. 单独发链接让它 unfurl
- 正文回复是流式 markdown 卡，里面的链接**不会**渲染成引用卡片。
- 额外单独发一条纯文本裸链接：
  ```bash
  # link.json = {"text":"https://my.feishu.cn/wiki/<wiki_token>"}
  lark-cli im +messages-send --chat-id <chatId> --msg-type text \
    --content "$(cat /tmp/link.json)" --as bot
  ```
- chatId 取自 `bridge_context.chatId`；一条消息只放一个链接最稳；正文里提一句「链接单独发你」，别把 URL 包进代码块 / 行内 code / `[标题](url)`。

### 5. 给用户结论
- 先一句话核心结论，再给文档结构概览。链接已单独发出。

---

## 已知空间 ID
- AI Related = `7647550775677226188`（默认归档目标）
- AI Builders Daily Newsletter = `7647334851355446238`

---

## 配图 / 画板
- 结构（架构 / 流程 / 分类 / 时间线）：用 **Mermaid 画板**（lark-whiteboard skill），比文字清楚。
- 产品 / 界面：抓**官方截图**随文附上。
- 能力盘点：每类配一张渲染示例图。
- 临时文件用完即清理。

---

## 踩坑速查表

| 症状 | 原因 | 解 |
|---|---|---|
| 文档标题是 Untitled | docxXML `<title>` 不生效 / overwrite 重置 | 建完显式 `update_title`，核验 |
| 文档不在知识库 | `docs +create` 默认落云盘 | 建完立即 `wiki +move` |
| 链接不渲染成卡片 | 流式 markdown 卡不 unfurl | 单独发 `msg_type:text` 裸链接 |
| shell 挂起 | 命令行出现中文 | 中文写文件再 `@file` 引用 |
| 文档改不了归属 | 用 user 身份建的 | 默认 `--as bot` 建 |
| 建文档失败/missing | user token 过期 | 用 bot 身份；或私聊重新 OAuth |
