---
name: batch-migration
description: 语雀知识库「全量一次性」批量搬运（千篇级）。触发词仅限：全量搬运、批量迁移、把整个库搬到、千篇级迁移、去剪藏垃圾样式、末尾加源链接、断点续传。特征：纯规则 R1-R8、无 LLM 分类、会清洗正文 + 注入源链接（≠ 原样搬）；不做逐篇筛选 / 边看边审 / 零转换搬运——那些走 doc-reorg。区别于 yuque-migration（脚本 migrate.py + LLM 去重）。
---

# 语雀全量批量搬运

复制不搬。源库**完全不动**。全量扫描、读取内容、清洗垃圾样式、源链接注入、断点续传。

## 核心原则

1. **复制不搬**：源库 A 库文档一律保留，不删除、不改动
2. **全量扫描**：默认全扫法（R5），不做「先列清单再审」，不靠 LLM 分类，纯规则 R1-R8 判定
3. **必读内容**：每篇必须 `yuque_get_doc` 读正文，禁止只用标题/`word_count` 元数据判断有用性
4. **内容清洗**：去剪藏垃圾样式（HTML 内联样式、平台导航栏、页脚、多余空白）——清洗的是同格式内噪音，**不改 format**
5. **源链接注入**：新文档末尾追加源文档链接，方便溯源
6. **断点续传**：每处理 20 篇保存一次进度到 JSON，中断后跳过已处理的 doc_id 续跑

## 与姊妹 skill 的边界

| skill | 方案 | 适用 |
|---|---|---|
| `doc-reorg` | 随看随搬、零格式转换、AI 逐篇判 | 中小库、老板边看边审 |
| `yuque-migration` | 脚本 `migrate.py` + LLM 分类 + 去重 + 重拟标题 | 需要智能去重/重命名的重型迁移 |
| **本 skill** | 纯规则全扫 + MCP 工具直驱 + 清洗 + 源链接 | 千篇级大批量、无 LLM、要求溯源 |

> **触发隔离**：只说「搬到」默认归 `doc-reorg`（零转换、最保守）；唯有明确出现「全量 / 批量 / 千篇级 / 整个库一次搬完 / 去剪藏垃圾 / 加源链接 / 断点续传」才切到本 skill。本 skill **不做**逐篇筛选、边看边审。
> 注：本 skill 会修改正文（清洗 + 注入源链接），不是原样搬，别拿它当零转换搬运使。

## 前置环境

- 语雀操作走 MCP（`yuque-mcp`，禁止直接 curl 调语雀 API）
- 列表：`yuque_web_list_docs`（默认裁剪输出含 `type`/`word_count`/`format`/`slug`；`raw=true` 才含 `editor_meta`）
- 读取：`yuque_get_doc`（按 format 选 body 字段）
- 写入：`yuque_create_doc`（body ≤ 50KB）/ `yuque_import_file`（body > 50KB；body 写本地文件，`paths` 必填）
- 目录：`yuque_get_toc` / `yuque_update_toc` / `yuque_batch_update_toc`

## 判定规则（复用 doc-reorg R1-R8）

> 判定优先级：R1 > R2 > R8 > R6 > R5 > R7 > R3，顺序判定，命中即止（先判「搬不搬」，再判「怎么搬」）。

| 规则 | 判定 |
|---|---|
| R1 正文二进制 | 纯二进制乱码 → 不搬；lake 含 `<card>` 标签不算二进制 |
| R2 数据库 dump | 不搬 |
| R3 附件 | 正文是文字的，附件随正文走 |
| R4 格式 | 源 `format` 原值写入（markdown/lake/html）零转换；body 字段按 format 选：`markdown`→`body`，`lake`→`body_lake`（用 `body` 会丢 card 附件），`html`→`body_html` |
| R5 有用性 | 默认全扫法，非以上拦截项全搬 |
| R6 结构化类型 | type=Sheet/Board/Table → 标记待老板裁决，不强行搬 |
| R7 Big Doc | body > 200KB 按 ~200KB 段落拆分；> 50KB 改走 `yuque_import_file` |
| R8 无意义内容 | 正文 < 10 字符或纯数字/标点/空白 → 不搬 |

> 编号与 `doc-reorg` 对齐（R1–R8），细则以 doc-reorg `references/rules.md` 为准。

## 流程（全量批量脚本）

写一个单线程串行循环，逐篇：

1. **拉清单**：`yuque_web_list_docs` 分页拉全部文档，展开 `word_count`、`type`、`format`、`slug`、`title` 字段
2. **元数据预过滤（不 fetch body，仅高置信项）**：
   - `word_count < 10` → R8 跳过（典型如批量「松建华」垃圾文档，几百篇一次全拦，省大量 API）
   - `type` 非 Doc → R6 跳过/标记
   - `word_count > 10000` 且标题为短字母数字组合（如 `a126713`）→ 二进制碎片特征，R1 跳过
   - ⚠️ 仅凭 `word_count` 大（如 >100000）**不得静默跳过**：会误杀小说/长教程/手册。必须 fetch body 确认，或标「待复核」列入报告（见核心原则 3：禁止只用元数据判断）
3. **取正文**：`yuque_get_doc` 按 format 选对 body 字段
4. **内容清洗**（详见下节）
5. **源链接注入**（详见下节）
6. **写入**：body ≤ 50KB 用 `yuque_create_doc`；> 50KB 写本地文件用 `yuque_import_file`（`format` 一律传源原值，markdown / lake / html 都支持）
7. **记日志**：记录 文档名 / 源格式 / 结果（成功/跳过/错误 + 原因）
8. **传进度**：每 20 篇写一次进度 JSON，支持断点续跑

## 内容清洗（去剪藏垃圾样式）

针对网页剪藏/论坛搬来的 markdown/lake 正文，清洗以下噪音（**不改 format**，清洗后仍传源 format 原值）：

| 清洗项 | 说明 |
|---|---|
| HTML 内联样式 | 去掉 `style="..."`、`class="..."`、`data-*` 等属性 |
| 平台导航栏 | 去掉 LINUX DO / 博客园 等页首导航、面包屑、侧栏 |
| 页脚 | 去掉「本文由 xxx 发布」「著作权归作者所有」等页脚 |
| 多余空白 | 压缩连续空行、去行首行尾空白 |

清洗后 body < 10 字符的按 R8 跳过（说明清洗后只剩垃圾）。

## 源链接注入

搬运后在新文档正文末尾追加（markdown/lake 通用）：

```markdown
---

> 本文档从 [源文档](https://www.yuque.com/{login}/{book_slug}/{doc_slug}) 搬运
```

- 链接来源：文档对象 `book.slug` + `slug` 字段拼接
- 登录名：`yehuoshun`（按实际语雀账号调整）

## Big Doc 处理

- body ≤ 50KB：`yuque_create_doc` 直接写
- body > 50KB：`yuque_create_doc` 会触发 `Argument list too long`（body 走命令行参数），改 `yuque_import_file`（body 写本地文件，命令只传文件路径）
- body > 200KB：按 **~200KB 段落**拆分——在段落/空行边界切分，每段 ≤200KB，每条用 `yuque_import_file`，标题 `{原文档名} - 第 N 段`；切分点优先选段落边界，避免掐断句子
- format：`yuque_import_file` 的 `format` 支持 `markdown` / `lake` / `html`，写入一律传源 format 原值（**含 lake，不转格式**）；lake 拆分点避开 `<card>` 块

## 按内容建 TOC

全量搬运后，目标库文档默认**平铺在根目录**（`yuque_create_doc` 不自动建目录），需按内容建目录树。

### 流程

1. **分类**：拉全量文档标题，按标题线索自动分（板块/作者/站点/扩展名）：
   - `标题 - 开发调优 - LINUX DO` → `LINUX DO/开发调优`
   - `标题 - 作者 - 博客园` → `博客园/技术文章`（或按作者细分）
   - 标题含 `.apk/.zip/.mp4` 等 → `游戏`
2. **建目录**：`yuque_update_toc` 逐层建 TITLE（`action=appendNode, action_mode=child, type=TITLE, title=X`，嵌套传 `target_uuid=父目录 uuid`）
3. **移文档**：`yuque_batch_update_toc` 批量 `moveNode`（`node_uuid`=文档当前 uuid，`target_uuid`=目标目录 uuid），`confirm` 必须传 `RESTRUCTURE`

### 关键坑

- **语雀 API 的 `target_uuid` 对 DOC 类型不可靠**，但本 MCP 已内部补偿：`appendNode` 带 `target_uuid` / `target_title` 时会自动「先挂根 → remove → append 到目标」；`moveNode` 等价于 `appendNode + node_uuid`。两者都可用，按 uuid 定位优先 `moveNode`
- **moveNode 会改文档 uuid**：移动某文档不影响其他文档 uuid，可先做「doc_id → uuid」快照批量移；每移一批后重取 `yuque_get_toc` 更稳
- **建目录用单工具 `appendNode` TITLE 会盲目追加**，同名目录重复建——先 `yuque_get_toc` 查已存在目录复用，事后清掉空重复目录
- **分块执行**：每批 50-60 个 `moveNode`，避免 ops 数组过大

## 断点续传

- 进度 JSON 结构：`{processed, success, skipped, errors, skip_reasons, results}`，`results` 记录每篇 doc_id + 结果
- 每次续跑：读进度文件 → 把已处理的 doc_id 加入集合 → 循环时跳过
- 写进度用「先写临时文件再 rename」避免中断写坏

## 执行报告 + 根目录首插

搬运结束强制出报告（概览 + 搬运清单 + 跳过清单），交老板终审，不自动发布。

报告放目标库/源库根目录**首位**时：

1. `yuque_import_file` 导入报告（`paths` 必填非空，先用临时目录名如 `["整理报告"]`）
2. `yuque_update_toc` 调到首位：`action=prependNode` + `action_mode=sibling` + `node_uuid`（报告节点）+ `target_uuid`（当前第一个根节点 uuid）
3. 若 import 生成了空临时 TITLE 目录：`action=removeNode` + `action_mode=sibling` + `confirm="DELETE"` 清掉

> 首插用**单文档** `yuque_update_toc` 的 `prependNode`（`action_mode=sibling` 保证同级插入），不是 `batch_update_toc` 的 `moveNode`（`moveNode` 不暴露 `position`，target 为 TITLE 时会变成其子级，控不了同级首插）。
> ⚠️ `yuque_update_toc` 的 `action_mode` 是必填参数（`sibling` / `child`），漏传会被参数校验拒掉。

## 批量脚本实现注意事项

- 子进程调 `mcporter` **必须指定 `cwd="/home/admin/.openclaw/workspace"`**，否则找不到 MCP 配置
- `yuque_get_doc` 的 `id` 参数必须是字符串
- `paths` 参数必须 JSON 数组字符串（`'["目录名"]'`），用 `--args` 传参
- 标题可能含 tab/换行，创建前 `re.sub(r'[\t\n\r]+', ' ', title)` 净化
- 单线程串行、禁止并发；处理完一篇释放 body 再处理下一篇（防 OOM）
- 单次操作超时 30 秒重试 1 次，仍失败跳过并记录

## 错误处理

- **429 限流**：v2 API 限流时渐进退避 1s/3s/5s，或改用 cookie 态 web 接口（`yuque_web_*`）
- **body 过大**：`Argument list too long` → 改 `yuque_import_file`
- **Big Doc 拆分失败**：标记「拆分失败」跳过并记录，不阻塞全量
- **拿不准**：R5 命中不了硬规则时打标「拿不准」列给老板，不搬