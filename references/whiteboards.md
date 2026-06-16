# 用画板把调研对象画清楚（默认动作，不是可选）

调研文档的价值不只在「写全」，更在**让读者快速、深度理解调研对象**。文字擅长叙述，**结构 / 对比 / 光谱 / 评分这类信息用画板一眼就懂**。所以：

> **黄金法则：每篇调研文档至少配 1 张、复杂对象配 3–5 张可编辑画板。**遇到「A vs B」「概念地图」「程度光谱」「打分表」「架构 / 数据流」「时间线」就上画板，别用大段文字硬讲。

本环境用两套能力配合：

- **`beautiful-feishu-whiteboard` skill**：提供 24 套配色风格 + 飞书 SVG 画板的硬规则。**你**来排版，它给你调色板和审美。
- **`lark-whiteboard` skill / `whiteboard-cli`**：把 SVG **或 Mermaid** 转成飞书 openapi 画板并写进文档、查询渲染图。

---

## 先选对工具：表达第一，美观第二

> **不必每张图都走 beautiful-whiteboard 的 SVG。先问「这块内容的本质是什么」，再选能最清楚表达它的工具——表达力第一，美观第二。**

| 内容本质 | 选什么 | 为什么 |
|---|---|---|
| **真实的流程 / 循环 / 时序 / 因果图 / 状态机**（有方向、有节点、有边） | **Mermaid**（经 `lark-whiteboard` 直接渲染） | 自动布局连线，画「环」「多跳回溯」「闭环」远比手画 SVG 箭头省事、不易错位 |
| **矩阵 / 卡片墙 / 程度光谱 / 计分表 / 分层架构栈 / 对照板** | **SVG**（`beautiful-feishu-whiteboard` 风格） | 需要精确版式、配色、并排卡片、色块强调——SVG 完全可控 |

判断口诀：**「这是一条会动的线，还是一张要摆好的表？」** 会动的线（流程/循环/因果）→ Mermaid；要摆好的表（卡片/矩阵/光谱）→ SVG。同一篇文档里两者可以混用，每张图各按本质选。

### Mermaid 画板工作流

```bash
# 1. 写 .mmd（节点文字 ≤8 字；步骤 ≤12；特殊字符用引号包：A["foo(bar)"]；subgraph 分组）
# 2. 本地渲染看图
npx -y @larksuite/whiteboard-cli@^0.2.11 -i diagram.mmd -o diagram.png
# 3. 灌进已插入的空白画板块（与 SVG 同一条 openapi 管线，只是输入换成 .mmd）
npx -y @larksuite/whiteboard-cli@^0.2.11 -i diagram.mmd --to openapi --format json \
| lark-cli whiteboard +update --whiteboard-token <block_token> --source - \
    --input_format raw --idempotent-token <unique> --overwrite --as bot --json
```

- Mermaid 节点文字**要短**（长标签会裁切），细节放进正文，别全塞进节点。
- 真实「环」（如 signal→search→…→sync→signal）用 `flowchart LR/TD` + 一条虚线回边 `A -. label .-> B` 表达自维护/夜间回流。
- 富信息流程图（很多分支 + 注释框）反而 SVG/DSL 更可控；Mermaid 最适合 sequence / class / pie / state / 简单 graph。

---

## 什么时候配什么图（调研专用映射）

| 调研里的内容 | 配什么画板 | 工具 |
|---|---|---|
| 两种范式 / 方案 / 产品对立 | 左右**对照板**（A vs B，中间放金句/结论条） | SVG |
| 3–6 个核心概念 | **概念卡地图**（每张卡：定义 + 一句「它在 X 里缺席 / 谁最接近」） | SVG |
| 程度 / 成熟度 / 自主性递进 | **光谱条**（低 → 高，把调研对象钉在某一格） | SVG |
| 多维度逐项评估 | **计分表**（维度 × 状态，✅/🟡/🔴 标记，附一句话理由） | SVG |
| 分层架构 / 模块栈 | **分层栈板**（L1…Ln 色块 + 融合列） | SVG |
| **流程 / 数据流 / 因果回溯 / 闭环循环 / 时序** | **流程图 / 循环图**（有向节点+边） | **Mermaid** |
| 阶段 / 演进 / 路线 | **时间线**（横向阶段卡） | SVG（强流程感时可 Mermaid） |

一张图只讲一件事；讲不清就拆成两张。**选 SVG 还是 Mermaid 看上面「先选对工具」一节。**

---

## 标准工作流（SVG → 写进文档 → 核验）

### 1. 设计 SVG（用 beautiful-feishu-whiteboard 的风格）
- 读 `beautiful-feishu-whiteboard/RULES.md` + 选定风格的 `templates/<slug>/design.md`。
- 画布约 1600–1700 宽，**native 形状**（rect / 圆角 rect / circle / line / text），每个标签是 `<text>`，**绝不设 `font-family`**。
- **只放内容，不放指令**：别把 prompt、来源、风格名、「summary of…」印到画布上——那像作业抬头。
- 用脚本批量生成多张时，**别用 bash 关联数组**（`declare -A` 在本环境 sh 里不支持，会静默退化），用 Python 生成器或逐条命令。

### 2. 本地渲染 → 看图 → 自我纠错
```bash
npx -y @larksuite/whiteboard-cli@^0.2.11 -i board.svg -o board.png -f svg
```
- **必须 view 渲染图**，修文字溢出、边距过紧、数字贴边、重叠、裁切。render→view→fix 循环到干净为止。

### 3. 在目标文档里插入空白画板块
先查到各小节标题的 block_id（`feishu_list_all_doc_blocks` 或 `docs +get`），在对应标题后插入空白画板，**逐条插入并单独捕获每个新 block_token**：
```bash
lark-cli docs +update --api-version v2 --doc <doc_id> \
  --command block_insert_after --block-id <heading_block_id> \
  --content '<whiteboard type="blank"></whiteboard>' --as bot --json
# 返回 data.document.new_blocks[0].block_token —— 这就是要填充的画板 token
```
⚠️ **不要在一个 bash 循环里用关联数组映射 heading→board**——失败时会全部插到同一个标题下。逐条插、逐条记 token 最稳。插错了用 `--command block_delete --block-id <id>` 删掉重来。

### 4. 把 SVG 灌进画板块
```bash
npx -y @larksuite/whiteboard-cli@^0.2.11 -i board.svg --to openapi --format json \
| lark-cli whiteboard +update --whiteboard-token <block_token> --source - \
    --input_format raw --idempotent-token <unique-token> --overwrite --as bot --json
```
- **必须用 stdin `--source -`**；传文件路径给 `--source` 会报 `invalid character '/'`。
- `--idempotent-token` 每次唯一（如 `name-$(date +%s)`）。

### 5. 核验线上渲染
```bash
lark-cli whiteboard +query --whiteboard-token <block_token> \
  --output_as image --output ./live_name --as bot   # 只能用相对路径
```
view 出来的图，确认线上和本地一致、位置对、没串行。

### 6. 收尾
- 重新发一次文档链接（text 消息 unfurl），一句话说明加了哪几张图。
- 清理 `/tmp` 临时文件（生成器、SVG、PNG）。

---

## 渲染踩坑速查

| 症状 | 原因 | 解 |
|---|---|---|
| ✓ / ✗ 显示成豆腐块 | Noto Sans SC 无该字形 | 用 `是/否` 或 emoji `✅ 🟡 🔴`（这些能渲染） |
| 字体不一致 / 报错 | 设了 `font-family` | 永远别设，单字体 |
| `--source` 报 invalid character | 传了文件路径 | 改用 stdin `--source -` |
| 4 张图全挤在一个标题下 | bash 关联数组退化 | 逐条 insert，逐条记 token |
| 画板写进去但渲染空 | openapi 转换失败 | 先确认 `--to openapi` 那步 stdout 是合法 JSON |
| Mermaid 节点文字被裁切 | 节点标签太长 | 节点 ≤8 字，细节放正文；步骤 ≤12 |
| Mermaid 报解析错 | 节点含括号/特殊字符 | 用引号包：`A["foo(bar)"]` |
| 命令行中文挂起 shell | hook 限制 | 中文写文件再 `@file`；画板文字在 SVG 里没事 |

---

## 配色与版式（沿用 beautiful-feishu-whiteboard）
- 整套文档的画板**用同一种风格**，视觉统一（例：Cobalt Glaze 全程）。
- 浅底深字；只有纯色强调块上才用浅色字。
- 不用渐变 / 滤镜 / 阴影 / 透明度（飞书画板不稳）。
- 箭头用 `marker-end="url(#arrow)"`。

> 风格是「调色板 + 气质」，排版永远由你掌控。换风格时同内容重渲染即可。
