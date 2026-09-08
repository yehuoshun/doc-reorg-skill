# yuque doc skills

语雀知识库文档整理 / 迁移技能合集。多套方案并存，按场景选用。

## 技能列表

### 1. doc-reorg — 随看随搬（轻量）

语雀知识库文档整理：**随看随搬、零格式转换、人工终审闸**。

- **适用**：中小库，老板边看边审，AI 逐篇判有用性
- **入口**：[`doc-reorg/SKILL.md`](doc-reorg/SKILL.md)
- **规则细则**：[`doc-reorg/references/rules.md`](doc-reorg/references/rules.md)
- **报告模板**：[`doc-reorg/references/report-template.md`](doc-reorg/references/report-template.md)

### 2. batch-migration — 全量批量搬运（重型）

纯规则 R1-R8 全扫 + MCP 直驱 + 内容清洗 + 源链接注入 + 断点续传。

- **适用**：千篇级大批量、无 LLM、要求原样 + 溯源
- **入口**：[`batch-migration/SKILL.md`](batch-migration/SKILL.md)

### 3. yuque-migration（独立仓库）

需要「批量清洗 + 智能去重 + 断点续传」+ LLM 分类去重？看 [yuque-migration-skill](https://github.com/yehuoshun/yuque-migration-skill)

## 方案对比

| skill | 方案 | 适用 |
|---|---|---|
| doc-reorg | 随看随搬、零转换、AI 逐篇判 | 中小库、边看边审 |
| batch-migration | 纯规则全扫 + 清洗 + 源链接 + 断点续传 | 千篇级大批量、无 LLM |
| yuque-migration（独立） | migrate.py 脚本 + LLM 分类 + 去重 + 重拟标题 | 需智能去重 / 重命名 |

## 协议

MIT License
