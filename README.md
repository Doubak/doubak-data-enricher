# doubak-data-enricher

豆备（Doubak）的数据关联器。

将用户从豆瓣（douban.com）本地备份的数据与 IMDb、MusicBrainz、Goodreads、Steam、Wikidata 等公开外部数据源建立映射与关联，并提供墓碑条目恢复与上游异 ID 重建合并能力。产出与 `bundle` 同样可独立备份、可完整带走的标准化增强归档包（`doubak-enrichment-<id>`），供后续建站或数据导出直接使用。

完整的产品需求与系统架构方案请参阅：
👉 **[PRD.md](PRD.md) —— 产品需求与架构设计说明书**

## 核心职责

1. **墓碑证据恢复（Tombstone Recovery）**：针对豆瓣上游彻底删除的条目（如原《情感反诈模拟器》`game/37364867`），通过历史抓取快照、Wayback Machine 快照、Wikidata 结构化链接及垂直数据库恢复标题、别名与海报。
2. **基于证据的条目对齐与合并（Evidence-based Entity Alignment）**：针对豆瓣删除条目后以新 ID 重建（如《捞女游戏》`game/33375066`）导致的条目割裂，基于全球唯一标识与页面元数据证据建立实体映射（`same_as`），向下游静态站点与导出适配器提供平滑的时间线与条目聚合。
3. **结构化元信息提取（Structured `raw_meta` Extraction）**：对解析器保留的未拆解 `intro` / `pub` / `desc` 斜杠字符串进行规则抽取，标注置信度与来源。
4. **语言与跨语言标识标注（Language & External IDs Tagging）**：为又名与标题标注 BCP 47 语言标签，关联全球通用实体 ID。

## 架构原则

- **便携式归档包标准**：与 `bundle` 对称，产物封装为自包含的 `doubak-enrichment-<id>` 归档包，含双语 `README.txt`、`manifest.json`、上层 NDJSON 与封存所有外部网络响应报文的标准 WARC 凭证段，可单包备份带走。
- **绝不允许主观手填元数据**：拒绝任何主观手工篡改标题、海报、年份的后门，所有数据增强必须来自公开可信、有据可查的证据链。
- **构建时零网络请求**：Enricher 发起的网络请求与响应在本地离线缓存；下游静态建站与导出适配器 100% 离线运行。
- **纯粹衍生缓存**：Enricher 产出独立存放，不污染 `canonical` 底层客观事实日志，随时可清空并重新推导。
- **零运行时依赖**：遵循 monorepo 统一标准，Node ≥ 20 原生 ESM 模块与 `node:test`。
