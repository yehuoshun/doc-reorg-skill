---
name: doc-reorg
description: 语雀知识库文档整理 Skill。当用户要求整理语雀文档、从 A 库筛选搬运有用文档到 B 库、知识库文档迁移、随看随搬、文档归档筛选时使用。核心：随看随搬 + 零格式转换 + 人工终审闸。
---

# Doc Reorg — 语雀知识库文档整理

把源知识库（A 库）里的有用文档筛选搬运到目标知识库（B 库），
随看随搬，格式零转换，人工终审闸兜底。

## 核心原则

1. **随看随搬**：文档量大时不做"先列清单再审"（老板审不过来），AI 边扫边判边搬
2. **搬运与格式解耦**：搬运阶段只判"有用性"，原样搬；格式统一（如有）单独一轮做
3. **零格式转换**：源文档 `format` 是什么就写什么（markdown / lake / html），不做格式转换
4. **内容清洗（可选）**：源文档来自剪藏/论坛时，去除垃圾样式（HTML 内联样式、导航栏、页脚、多余空白）。**清洗 ≠ 格式转换**——只清同格式内的噪音，不改 format
5. **源链接注入（可选）**：搬运后在新文档末尾追加源文档链接，方便溯源
6. **不删源文档**：搬运是复制语义，源库文档一律保留、不删除、不改动
7. **人工终审闸**：AI 自查 ≠ 过审，过审以老板/主管人工终审为准，终审通过前禁入库禁发布

## 触发场景

- "整理语雀 / 整理 X 库到 Y 库"
- "从 A 库找有用文档搬到 B 库"
- "语雀文档迁移 / 筛选 / 归档"
- "随看随搬"
- "全量搬运 / 去除剪藏垃圾样式 / 末尾加源链接"

## 前置环境

- 语雀操作必须走 MCP（`yuque-mcp`，禁止直接 curl 调语雀 API）
- 列表工具：`yuque_web_list_docs`（分页拉取，返回 editor_meta 字段，可用于快速预判附件文档）
- 读取工具：`yuque_get_doc`（v2 API，返回完整 body / body_lake / body_html）
  - ⚠️ `yuque_web_get_doc`（web API）不返回 body_lake，仅用于轻量查询
- 搬运工具：`yuque_create_doc`（写入目标库，format 传源格式，body 传对应字段）
- 目录结构：`yuque_get_toc` / `yuque_batch_update_toc`

## 判定规则（先手拦截，AI 严格照办）

| 规则 | 判定 |
|---|---|
| R1 正文二进制 | 正文是纯二进制乱码/不可读内容 → **不搬**。lake 格式含 `<card>` 标签的文档不算二进制（即使标题含文件扩展名，如 `.w3x/.js/.gif`，也不跳过） |
| R2 数据库 dump | 内容是数据库 dump → **不搬** |
| R3 附件 | 正文是文字的，附件照搬，随正文走 |
| R4 格式 | 源 `format` 是什么就写什么（markdown/lake/html）零转换；`yuque_get_doc` 返回的 body 字段选择：`format=markdown` → `body`，`format=lake` → `body_lake`，`format=html` → `body_html` |
| R6 类型 | `type=Sheet/Board/Table` 结构化文档另案处理（copy_doc 传不了结构化正文） |
| R7 Big Doc 拆分 | 文档 body > 200KB → 按章节拆分搬运；body > 50KB 时 `yuque_create_doc` 会触发 `Argument list too long`，改用 `yuque_import_file`（body 写入本地文件，命令只传文件路径） |
| R8 无意义内容 | 正文极短（< 10 字符）或仅含无意义字符（纯数字/标点/空白/对象引用/JSON元数据）→ **不搬** |
| R5 有用性 | 命中"有用范围"才搬；默认**全扫法**（非二进制/非dump/非结构化文档 的全搬） |

> 判定标准只看 body 内容，不看标题。
> **优化技巧**：`yuque_web_list_docs` 返回的 `editor_meta` 字段可快速判断文档是否含附件（`{"file":N}` / `{"video":N}`），有 editor_meta 的文档一定是合法 lake 文档，无需调 get_doc 确认。纯二进制上传碎片的 editor_meta 为 null。
> 标题含 `.7z` / `.flv` / `.mp4` / `.zip` / `.rar` 等扩展名可以作为快速预判参考，但不能作为跳过依据——必须获取 body 后确认是否纯二进制乱码。
> lake 格式文档即使 body 含不可读数据（如视频卡片、文件卡片嵌入），只要包含 `<card>` 标签，就不算二进制。
> 判定优先级：R1 > R2 > R8 > R6 > R7 > R5 > R3，顺序判定，命中即止。

## 内容清洗与源链接注入（可选能力）

> 老板明确要求"去剪藏垃圾样式"或"末尾加源链接"时才启用；默认仍为零转换（原样搬）。

### 内容清洗（去剪藏垃圾样式）

针对从网页剪藏/论坛搬来的 markdown/lake 正文，清洗以下噪音（**不改 format**）：

| 清洗项 | 说明 |
|---|---|
| HTML 内联样式 | 去掉 `style="..."`、`class="..."`、`data-*` 等属性 |
| 平台导航栏 | 去掉 LINUX DO / 博客园 等页首导航、面包屑、侧栏 |
| 页脚 | 去掉「本文由 xxx 发布」「著作权归作者所有」等页脚 |
| 多余空白 | 压缩连续空行、去行首行尾空白 |

清洗方式：拿到 body 后先清洗再搬运；`format` 仍传源文档 format 原值。清洗后 body < 10 字符的按 R8 跳过。

### 源链接注入

搬运后在新文档正文末尾追加（markdown/lake 通用）：

```markdown
---

> 本文档从 [源文档](https://www.yuque.com/{login}/{book_slug}/{doc_slug}) 搬运
```

### 不删源文档

搬运是**复制**语义，源库 A 库文档一律保留，不删除、不改动。

## 流程

```mermaid
flowchart TD
    A[取一篇A库文档] --> B{正文是纯二进制乱码?<br/>body 无可读文本<br/>且无 lake card 标签}
    B -- 是 --> X[不搬]
    B -- 否 --> N{无意义内容?<br/>正文<10字符<br/>或纯数字/标点/空白}<br/>N -- 是 --> X
    N -- 否 --> C{是数据库dump?}
    C -- 是 --> X
    C -- 否 --> T{type 是 Sheet/Board/Table?}
    T -- 是 --> W[标记待老板裁决<br/>不强行搬]
    T -- 否 --> G{body > 200KB?}
    G -- 是 --> H[下载文档<br/>按章节拆分]
    H --> I[分别搬运各章节<br/>标题: 原文档名 - 章节名]
    G -- 否 --> D{属于有用范围?}
    D -- 是 --> E[原样搬进B库<br/>保留 format 原值<br/>markdown/lake/html 零转换<br/>附件跟着正文走]
    D -- 否 --> X
    E --> F[记录搬运日志]
    I --> F
```

### 步骤

1. **定范围**：向老板确认 A 库、B 库、以及"有用范围"（范围法 / 目录法 / 全扫法，默认全扫法）
2. **列文档**：`yuque_web_list_docs` 分页拉取 A 库文档
3. **逐篇判定**：按 R1→R2→R6→R7→R5 顺序判定，另查 `type` 字段（R6）
4. **取正文**：`yuque_get_doc` 获取文档后，按 format 选择对应 body 字段：`format=markdown` → `body`，`format=lake` → `body_lake`，`format=html` → `body_html`。**禁止**用 `body` 字段搬运 lake 格式文档（会丢失 card 标签内的附件链接）
5. **小文档搬运**：`yuque_create_doc` 写入 B 库，`format` 传源文档的 format 原值，`body` 传上一步选中的正确 body 字段
6. **Big Doc 拆分**：`yuque_get_doc` 获取 body，按章节标题（`#`/`##`/`###` 或 `一、二、三` 等中文编号）拆分，每条用 `yuque_import_file` 写入 B 库（避免命令行参数长度限制），标题格式 `{原文档名} - {章节名}`。拆分后每篇保持源 format。纯文本无章节结构的文档降级为整体搬运（不做硬拆分）
7. **记日志**：记录 文档名 / 源位置 / format / 搬运结果，形成搬运日志
8. **出执行报告**：扫描结束生成报告，含概览 + 搬运成功清单 + **跳过清单（跳过原因 + 跳过文档链接）** + Big Doc 拆分清单 + 拿不准清单，模板见 `references/report-template.md`
9. **交终审**：报告 + 搬运结果交老板人工终审
10. **格式校验**：抽查 B 库文档确认格式未损坏（零转换后主要是校验动作）

### 执行报告（强制，每轮必出）

扫描结束必须产出一份执行报告，包含三块核心：

| 块 | 内容 | 格式要求 |
|---|---|---|
| 概览 | 扫描 / 搬运 / 跳过 / Big Doc 拆分 / 拿不准数量 | 数字 + 一眼可读 |
| 搬运清单 | 成功搬入 B 库的文档 | 文档名 + 源格式 + 目标位置 |
| **Big Doc 拆分清单** | 因 >200KB 被拆分的文档 | 原文档名 + 拆分章节数 + 目标位置 |
| **跳过清单** | 被规则拦截的文档 | **文档名 + 跳过原因 + 文档链接，每条都有** |
| 拿不准清单 | 硬规则覆盖不到的 | 文档名 + 原因 + 链接，交老板扫一眼 |

**跳过清单必须满足**：
- 每条跳过都写明原因（映射 R1 正文二进制 / R2 数据库dump / R5 不在有用范围）
- 每条跳过都带**文档链接**（`https://www.yuque.com/{login}/{book_slug}/{doc_slug}`），老板点开即可复核
- 不允许只报数量不报明细

> 示例链接格式：`https://www.yuque.com/yehuoshun/{book_slug}/{doc_slug}`
> 链接来源：文档对象里的 `book.slug` + `slug` 字段拼接

### 报告放置

- 报告放在**目标库或源库**（按老板指定）**根目录**（level=0），不嵌套子目录
- 创建方式：生成完整 markdown 内容到本地文件，用 `yuque_import_file` 导入（避免 API body 长度限制）
- **首插（可选）**：老板要求报告放根目录**首位**时，三步走：
  1. `yuque_import_file` 导入（`paths` 必填非空，先用临时目录名，如 `["整理报告"]`）
  2. `yuque_update_toc` 调 `action=prependNode` + `node_uuid`（报告节点 uuid）+ `target_uuid`（当前第一个根节点 uuid）提为首位
  3. 若 import 时生成了空的临时 TITLE 目录，用 `action=removeNode` + `confirm="DELETE"` 清掉
- 首插必须用**单文档** `yuque_update_toc` 的 `prependNode`，不是 `batch_update_toc` 的 `moveNode`（后者 position=before 配合 TITLE target 会把节点变子级而非同级）

### 跳过清单条目过多时的处理

当跳过条目超过 100 条时：
- 按原因分组展示（如 `R1 正文二进制（标题含 .7z）` 为一组）
- 每组首行标注数量，每条带链接
- 将完整 markdown 文件写入本地临时目录，再用 `yuque_import_file` 导入为目标库的首篇文档
- 完整 JSON 报告留存在本地供后续参考

模板见 `references/report-template.md`

## 人工终审闸（强制）

- AI 无权自判"审核通过"
- 终审通过前禁入库禁发布
- 终审通过后，搬运日志标注"(审核通过稿)"

## TOC 操作局限

- `yuque_batch_update_toc` 的 `moveNode` 操作中，`position=before` 配合 TITLE 类型的 `target_uuid` 时，**始终将目标节点变为该 TITLE 的第一个子节点**，而非同级插入
- 如需将文档放在根目录，直接用 `yuque_create_doc` 或 `yuque_import_file` 创建，不依赖 TOC 移动操作
- 不需要对目标库做复杂的 TOC 结构调整

## 批量脚本执行注意事项

> **量大优先用批量脚本**：文档上千篇时，写个循环脚本（逐篇 get_doc → 清洗 → create_doc），单线程串行处理，比 AI 逐篇人工判定快得多。配合进度持久化可断点续跑。以下为脚本实现的坑点：

- 子进程调用 `mcporter` 时，**必须指定 `cwd="/home/admin/.openclaw/workspace"`**，否则找不到 MCP 服务器配置
- `yuque_get_doc` 的 `id` 参数必须是字符串，即使传数字也会被 MCP 校验拒绝
- `yuque_copy_doc` 的 `paths` 参数必须是 JSON 数组字符串（如 `'["目录名"]'`），需用 `--args` 方式传参
- **命令行参数长度限制**：`yuque_create_doc` / `yuque_copy_doc` 将 body 作为命令行参数传递，body > 50KB 时可能触发 `Argument list too long` 错误。**解决方案**：body > 50KB 的文档用 `yuque_import_file` 替代（body 写入本地文件，命令只传文件路径）
- **word_count 预过滤（双向）**：先通过 `yuque_list_docs` 拉取全量文档的 `word_count` 字段，不 fetch body 即可拦截：
  - `word_count < 10` → 直接 R8 跳过（无意义纯水文档，如批量「松建华」垃圾文档，一次可拦截数百篇，省大量 API 调用）
  - `word_count > 100000` → 直接 R1 跳过（二进制碎片）
  - `word_count > 10000` 且标题为短字母数字组合（如 `a126713`）→ 大概率二进制碎片，直接 R1 跳过
- **路径标题净化**：`yuque_copy_doc` 的 `paths` 参数中的标题可能含 tab、换行等特殊字符，导致 JSON 解析失败。搬运前需 `re.sub(r'[\t\n\r]+', ' ', title)` 净化
- **批量处理超过 100 条时**，建议先 title 检测跳过二进制文件，再逐条 fetch body 验证
- **进度持久化**：每处理 20 条保存一次中间结果到 JSON 文件，支持断点续跑（跳过已处理的 doc_id）

## 错误处理

- **限流 429**：v2 API 被限流时改用 cookie 态 web 接口（`yuque_web_*`）
- **拿不准**：R5 命中不了硬规则时，打标"拿不准"列一行给老板扫一眼，不搬
- **Big Doc 下载失败**：标记为"下载失败"，跳过该文档，记录原因
- **Big Doc 拆分失败**：标记为"拆分失败"，跳过该文档，记录原因
- **OOM 预防**：禁止并发读取/写入，单线程逐篇处理；处理完一篇释放 body 后再处理下一篇
- **网络超时**：单次操作超时 30 秒，超时重试 1 次，仍失败则跳过并记录