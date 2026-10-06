# doubak-data-enricher

豆备（Doubak）的数据关联器。

将用户从豆瓣（douban.com）本地备份的数据与 IMDb、MusicBrainz、Goodreads、Steam、Wikidata 等公开外部数据源建立映射与关联，并提供墓碑条目恢复与上游异 ID 重建合并能力，关联结果缓存在本地，供后续建站或数据导出直接使用。

完整的产品需求与系统架构方案请参阅：
👉 **[PRD.md](PRD.md) —— 产品需求与架构设计说明书**

## 核心职责

1. **墓碑条目挽救（Tombstone Recovery）**：针对豆瓣上游彻底删除的条目（如《情感反诈模拟器》`game/37364867`），通过 Wayback Machine 快照、Wikidata 结构化链接及垂直数据库恢复标题、别名与海报。
2. **条目对齐与合并（Entity Alignment & Merging）**：针对豆瓣删除条目后以新 ID 重建导致的条目割裂，建立实体层映射（`same_as`），向下游静态站点与导出适配器提供平滑的时间线与条目聚合。
3. **结构化元信息提取（Structured `raw_meta` Extraction）**：对解析器保留的未拆解 `intro` / `pub` / `desc` 斜杠字符串进行规则抽取，标注置信度与来源。
4. **语言与跨语言标识标注（Language & External IDs Tagging）**：为又名与标题标注 BCP 47 语言标签，关联全球通用实体 ID。

## 架构原则

- **构建时零网络请求**：Enricher 发起的网络请求与响应在本地离线缓存；下游静态建站与导出适配器 100% 离线运行。
- **纯粹衍生缓存**：Enricher 产出独立存放，不污染 `canonical` 圣神观测，随时可清空并重新推导。
- **零运行时依赖**：遵循 monorepo 统一标准，Node ≥ 20 原生 ESM 模块与 `node:test`。
