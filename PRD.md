# doubak-data-enricher 产品需求与架构设计说明书 (PRD)

> **文档状态：** 架构设计定稿 (Approved Architecture Baseline)  
> **面向仓库：** `doubak-data-enricher`  
> **适用版本：** `enrichment/1.0`  
> **关联规范：** `doubak-data-specs` (canonical/1.1, bundle/1.4, enrichment/1.0), `doubak-site-generator`, `doubak-export-adapters`

---

## 0. 摘要与设计宗旨 (Executive Summary & Philosophy)

### 0.1 项目定位
`doubak-data-enricher`（豆备数据关联器）是豆备数据处理流水线中的**外部知识库关联与数据修补层**。它介于底层的规范归档解析器（`doubak-data-parser`）与上游的消费端（`doubak-site-generator`、`doubak-export-adapters`）之间。

```
[原始抓取 WARC Bundles]
           │
           ▼
[doubak-data-parser] ──(纯函数、离线、客观观测)──> [canonical/ 底层客观事实日志]
                                                         │
                                                         ▼
                                             [doubak-data-enricher] <──(外部公开可信数据源)
                                             (可选、带证据溯源、带置信度、本地离线缓存)
                                                         │
                                                         ▼
                                  [doubak-enrichment-<id>/ 便携式增强归档包]
                                  (可单包备份带走、含完整 WARC 证据与上层 NDJSON)
                                                         │
                                ┌────────────────────────┴────────────────────────┐
                                ▼                                                 ▼
                    [doubak-site-generator]                           [doubak-export-adapters]
               (投影合并 → Markdown → 独立网站)                     (无缝迁移至 NeoDB / Letterboxd / Goodreads)
```

### 0.2 为什么必须设计独立的 Enricher
豆瓣作为一个中心化平台，其目录数据存在天然的脆弱性：
1. **上游条目被彻底抹除（Tombstone / 墓碑条目）**：政治审查、商业纠纷或版权到期会导致条目从豆瓣库中彻底下架。用户个人标记虽在，但作品标题沦为「未知游戏/未知电影」、封面变成默认占位图、详情页 404。
2. **上游删除后异 ID 重建（Entity Drift & Recreation）**：条目被删除一段时间后，网友或官方可能以全新 ID 重新提交收录。此时同一现实作品在豆瓣历史上存在两个不相交的 ID，导致历史标记与新条目割裂。
3. **元数据字符串混杂未分拆（Opaque `raw_meta`）**：依据 `FIELDS.md` §4，列表页提取的 `intro` / `pub` / `desc` 是未打标签的斜杠分隔串（电影段数多达 43 种）。解析阶段严禁猜测，拆解必须移交 Enricher 标注处理。
4. **缺乏全球实体对齐与语言标注**：豆瓣的「又名」未标注语言（简/繁/粤/英/日/韩）；缺乏国际公认唯一标识（Wikidata QID、IMDb tt、Steam AppID、ISBN、MusicBrainz MBID），限制了向分布式社交网络与第三方平台的迁移质量。

### 0.3 核心设计铁律 (Architectural Invariants)
本仓库的架构与工程设计必须无条件恪守以下六条铁律：

1. **客观事实与派生缓存分离 (Immutable Facts vs. Derived Cache)**  
   WARC 原始捕获与用户标记是不可撼动的客观事实；`canonical` 是客观观测事件日志。Enricher 产出的所有外部 ID、推断元数据及恢复信息**纯属衍生缓存（Derived Cache）**。清空 Enricher 产出，整个系统依靠原始捕获依然能离线全量构建。
2. **和 Bundle 一样可独立备份、可完整带走 (First-Class Portability)**  
   Enricher 的产物绝不能是散落于本机 scratchpad 的临时缓存，而必须是一个**独立自包含、有版本清单、可打包压缩带走（Portable）、可离线冷备的一等公民归档包 (`doubak-enrichment-<id>`)**。无论是拷贝至 U 盘还是备份至 NAS，十几年后解压依然立即可用。
3. **条目丢失优先处理与渐进式固化 (Lost-First Priority & Progressive Commit)**  
   外部知识库接口均有严格限流与速率控制。系统绝不盲目线性遍历几千条作品，而是**强制实行优先级队列**：优先调度上游已删除的墓碑条目与残缺条目，且 P0 墓碑处理完毕后立即固化落盘。即便任务中途因网络中断或被用户终止，最关键的丢失条目已经成功恢复。
4. **绝不允许主观手填元数据，一切增强必须基于可信证据链 (No Manual Metadata Injection)**  
   严禁提供主观手工篡改标题、海报、年份等元数据的后门。手填元数据不仅无法保证真实性，还会引入不可维护的第二混乱真相源。Enricher 的所有数据补充，**必须来自有据可查、可重复验证的公开源**（历史抓取快照、Wayback Machine、Wikidata、Steam、TMDB、Bangumi 等），所有原始网络报文完整存入包内的标准 WARC 文件中封存。
5. **构建与渲染阶段绝不发起网络请求 (Zero Network at Build/Render Time)**  
   Enricher 是整个流水线中**唯一**被允许发起外部网络请求的组件。一旦抓取完成，所有关联结果及网络原始响应必须**固化落盘在包内**。静态站点生成（`site-generator`）与向第三方导出（`export-adapters`）必须永远保持 100% 离线运行。
6. **极简审计性与零运行时依赖 (Zero Runtime Dependencies)**  
   遵循 Doubak monorepo 统一工具链：Node ≥ 20，纯原生 ES 模块（ESM），JSDoc 类型标注，`node:test` 单测框架，零第三方 npm 运行时依赖，零构建步骤。

---

## 1. 核心问题剖析与真实基准案例 (Problem Statement & Benchmark Cases)

### 1.1 贯穿全篇的真实案例：从《情感反诈模拟器》到《捞女游戏》

为了确保设计的每一步都立足于真实数据，全篇以真实归档中刚刚捕获的完整生命周期作为基准案例：

* **背景**：互动叙事游戏《情感反诈模拟器》（Steam 英文名：*Revenge on Gold Diggers*，民间亦称《捞女游戏》）。
* **第一阶段：2025 年的正常标记与后续下架（老条目 `37364867` 沦为墓碑）**：
  * 用户于 2025-07-02 发广播「想玩」，2025-07-19 标记「玩过」并写下评语：
    * 评分：4 星（广播冻结）
    * 标签：`["中国", "游戏", "文字冒险", "益智", "互动电影"]`
    * 短评：*“游戏做的很用心，还有教学档案，其实也很适合女生玩。这游戏要是10多年前出来就好了，那时候身边好多男PUA (Pick-Up Artist)特别会撩妹，其实也就是这些技巧。”*
  * 2026 年 2 月底，该游戏在豆瓣因题材争议被官方彻底删除。
  * **Canonical 真实观测状态**：
    `subjects.ndjson` 中 `id: "37364867"` 变成墓碑条目：`title: null`, `aliases: null`, `info: null`, `upstream_deleted: true`, 海报退化为通用占位图 `game_normal.png`。
* **第二阶段：2026 年 10 月以新 ID 重建与用户重标（新条目 `33375066`）**：
  * 豆瓣以全新条目 ID **`33375066`** 重新收录该游戏，标题更新为《捞女游戏 Revenge on Gold Diggers》。
  * 用户在最新批次（`doubak-bundle-20261006T221030Z-e8a36c`）中重新将其标记为「玩过」：
    * 标记日期：`2026-10-06`
    * 评分：4 星
    * 标签：`["2025", "中国", "互动电影", "情感"]`
    * 真实短评：*“我真的无语，我很确定以前还叫《情感反诈模拟器》的时候就标记过，豆瓣删条目又重建条目了看来。当时我还说里面一些恋爱小贴士做的还不错的。不过anyway，这部其实制作还挺精良的”*
  * 新条目详情页特征：
    * URL: `https://www.douban.com/game/33375066/`
    * 标题: `捞女游戏 Revenge on Gold Diggers`
    * 海报: `https://img9.doubanio.com/lpic/s35470134.jpg`
    * 关键字 (Keywords): `捞女游戏, Revenge on Gold Diggers, 情感反诈模拟器, 天下无捞, PC, Mac, Linux...`

```
2025-07-19  用户标记 37364867 (情感反诈模拟器) ───┐
                                                  │ (2026-02 豆瓣下架删除，37364867 变成墓碑)
                                                  ▼
2026-10-06  用户重标 33375066 (捞女游戏) ─────────┴─> 【两份分离的客观观测！现实中是同一部作品】
```

* **痛点现状**：
  1. **Canonical 层必须保持分离**：根据 `IDENTITY.md` §2.4，`37364867` 与 `33375066` 是两个不同时期、不同 ID 的客观观测，解析器绝不能将其合并成一个，否则就破坏了历史观测的真实性。
  2. **静态站点上的分裂体验**：若无 Enricher，站点会生成两个孤立页面 —— `game/37364867.html`（叫“未知作品”，带着 2025 年的旧短评）与 `game/33375066.html`（叫“捞女游戏”，带着 2026 年的吐槽），历史脉络彻底断裂。
  3. **导出适配器上的残缺**：向 NeoDB 导出时，旧标记 `37364867` 因上游 URL 为 null 只能被抛弃在 `neodb-needs-check.csv`，而新标记 `33375066` 缺乏 2025 年那段完整的历史演进。

---

## 2. 系统功能规范 (Functional Specifications)

`doubak-data-enricher` 拒绝任何形式的主观手工编造数据，所有能力均基于**确定性的机器规则、可信凭证与证据链**。系统分为五大核心模块：

```
                    ┌────────────────────────────────────────────────────────┐
                    │                 doubak-data-enricher                   │
                    └────────────────────────────────────────────────────────┘
                                                 │
         ┌───────────────────┬───────────────────┼───────────────────┬───────────────────┐
         ▼                   ▼                   ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ 1. 丢失优先调度 │ │ 2. 墓碑证据恢复 │ │ 3. 实体证据对齐 │ │ 4. 元信息提取器 │ │ 5. 语言标记器   │
│ (Priority Queue)│ │ (Tombstone Rec) │ │ (Entity Align)  │ │ (RawMeta Parser)│ │ (Lang Tagging)  │
└─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘
  ├─ P0~P4 分级评估   ├─ 历史快照比对     ├─ 全球唯一标识对齐 ├─ 封闭词典比对     ├─ Unicode 字符集
  ├─ 渐进式提前落盘   ├─ Wayback CDX API  ├─ 页面关键词佐证   ├─ 格式模式匹配     ├─ 简繁粤英归类
  └─ --only-lost 参数 └─ Wikidata / Steam └─ 用户上下文佐证   └─ #info 表格比对   └─ BCP 47 编码标注
```

---

### 2.1 模块一：条目丢失优先的排队调度系统 (Priority Scheduling System)

一个真实用户的个人归档往往包含数千条（如本例 2984 条）甚至上万条作品。外部公共知识库（Wikidata SPARQL、Wayback Machine、Steam）均有严苛的速率限制与反爬防风控机制（通常需间隔 1.5 秒以上）。如果按物理文件行序盲目线性遍历，处理全量归档需要数十分钟甚至数小时。

然而，在这数千部作品中，**用户感知最剧烈、数据损坏最严重的，恰恰是那不到 1% 的上游丢失条目（当前仅 8 部）！** 健康存活的条目，静态站点原本就能正常显示标题与介绍；只有丢失条目才会沦为刺眼的“未知作品”，并在导出 NeoDB 时被直接丢弃。

因此，系统建立**五级优先级评估与调度模型 (P0 ~ P4)**：

#### 2.1.1 五级优先级队列模型

| 优先级 | 判定条件 | 核心抢救目标 | 数量占比 | 执行时机与策略 |
|---|---|---|---|---|
| **P0（致命级）<br>墓碑与丢失条目** | `upstream_deleted === true` 或 `title === null` 或 `url === null` | 优先通过 Wayback、Wikidata、Steam 抢救恢复真实标题、封面与全球 ID，避免在导出时被丢弃 | ~0.3%<br>(当前 8 部) | **最先执行**。通常仅需 3~5 秒即可全部完成，处理完毕后**立即固化落盘**。 |
| **P1（结构级）<br>疑似删除重建条目** | 新条目 keywords 载有旧条目名、或用户短评显式指出更名重现、或命中同一外部唯一 ID | 优先生成 `entities.ndjson` 聚合映射，打通跨 ID 断裂的历史时间线 | ~0.5% | P0 之后立即执行，确立核心作品聚类实体。 |
| **P2（残缺级）<br>关键展示项缺失** | 存活正常，但缺少海报（占位图）、或缺少详情页 `#info`、或影视缺少 IMDb 编号、或图书缺少 ISBN | 补齐 IMDb/ISBN 与高清海报，提升向 Letterboxd / Goodreads 导出的成功率 | ~2.0% | 第二阶段执行，集中修复影响外部平台导出的短板。 |
| **P3（创作级）<br>用户心血活跃条目** | 存活正常，但用户撰写了长文日记（`longform.ndjson`）、评语大于 100 字、多次打星或广播互动 | 优先为其提取结构化 `raw_meta`，丰富作品页详情呈现 | ~15.0% | 第三阶段执行，优先服务用户倾注心血最多的作品。 |
| **P4（闲时级）<br>常规普通条目** | 存活正常、元数据完整、仅点选“看过”且无独立评语 | 纯本地 CPU 跑 `raw_meta` 启发式提取与语言标注，不耗费外部 API 额度 | ~82.0% | 最末执行。可在后台批处理或增量闲时运行。 |

#### 2.1.2 渐进式提前落盘 (Progressive Early Commit)
* 当 P0（墓碑丢失条目）与 P1（对齐条目）处理完毕后，调度器**立即触发首次归档固化**：将恢复出的数据生成第一批 NDJSON 记录，并将网络凭证段写入 WARC 文件。
* **容错价值**：即便全量任务在处理后续 P3/P4 过程中遭遇网络波动、API 配额耗尽或被用户手动中断（Ctrl+C），**最重要的墓碑数据已经 100% 挽救落盘**，产出的归档包已经可以直接供下游建站与导出使用！

#### 2.1.3 CLI 快速拯救模式
提供细粒度 CLI 参数控制：
```sh
# 极速拯救模式：只扫描和抢救 P0（丢失条目），3~5 秒内完成并产出归档包
doubak-enrich <canonical> [out] --only-lost

# 针对性处理高优先级队列（P0 + P1 + P2）
doubak-enrich <canonical> [out] --priority=P0,P1,P2
```

---

### 2.2 模块二：墓碑证据恢复引擎 (Tombstone Recovery Engine)

针对 P0 墓碑条目，必须通过**客观凭证链**找回真实元数据，拒绝任何“凭空捏造”：

#### 2.2.1 凭证链恢复流水线

1. **Tier 1: 本地跨版本/历史快照回溯 (Local Archive Traceback)**  
   * **原理**：检查本地历史所有 bundle 中，是否在该条目被删除前曾抓到过其详情页。  
   * **实测成果**：真实归档中，`movie/11611021`《在这世界的角落》与 `game/24299254`《瘟疫公司》即在此层 100% 恢复。  
   * **置信度**：`source: "local_history:bundle_id"`, `confidence: 1.0`。

2. **Tier 2: Wayback Machine CDX API 历史快照提取**  
   * **原理**：针对豆瓣原始 URL（如 `www.douban.com/game/37364867/`）或广播中捕获的短链（如 `douc.cc/2GLwai`），请求 Internet Archive CDX API：  
     `https://web.archive.org/cdx/search/cdx?url=www.douban.com/game/37364867/&output=json&filter=statuscode:200`
   * **解析**：下载最近一次状态正常的存档快照，重用解析器的原生抽取逻辑提取当时的 `<title>`、`#info` 与海报。原始网络响应完整写入归档包的 WARC 文件中。  
   * **置信度**：`source: "wayback:<timestamp>"`, `confidence: 0.95`。

3. **Tier 3: Wikidata 结构化属性精准检索 (SPARQL)**  
   * **原理**：通过 Wikidata 登记的权威属性反向检索：  
     * 游戏：`wdt:P11867` (Douban Game ID)  
     * 影视：`wdt:P4438` (Douban Movie ID)  
     * 图书：`wdt:P11868` (Douban Book ID)  
   * **产出**：通过属性关联，获取官方多语言名、Steam AppID (`P1733`)、IMDb ID (`P345`)。网络响应写入 WARC 凭证段。  
   * **置信度**：`source: "wikidata:Q..."`, `confidence: 0.90`。

4. **Tier 4: 垂直领域官方数据库比对 (Steam / TMDB / Bangumi)**  
   * **原理**：当获得确定性的外部 ID（如 Steam AppID `3057160`）时，调用官方 API 拉取经过数字签名的权威官方元数据与封面。原始 JSON 与海报图片二进制完整存入 WARC 凭证段。  
   * **置信度**：`source: "steam:3057160"`, `confidence: 0.98`。

---

### 2.3 模块三：基于证据的实体对齐与条目合并 (Evidence-based Entity Alignment)

针对豆瓣上游“删除后换 ID 重建”的问题，Enricher 建立独立的实体对齐层，**不修改 canonical 历史，而是在上层产出聚合映射 (`entities.ndjson`)**。

#### 2.3.1 自动对齐的四大证据准则 (Zero-Guesswork Alignment Rules)
为了杜绝误合并，两个豆瓣条目（如 `37364867` 与 `33375066`）被判定为同一现实作品，必须满足以下**至少一条可信证据**：

1. **证据 A：外部全球唯一标识一致（Strong External ID Identity）**  
   * 两个条目经知识库检索后，对应相同的国际公认标识（如拥有完全相同的 Steam AppID `3057160`，或相同的 ISBN、IMDb ID、Wikidata QID）。
2. **证据 B：上游详情页的别名/关键词强包含（Upstream Metadata Keyword Link）**  
   * 新条目的页面元数据中明确载有旧条目的名称。  
   * **实测案例**：在 `game/33375066` 的真实详情页中，`<meta name="keywords">` 显式记录了 `情感反诈模拟器`（正是旧条目 `37364867` 的原名），这构成了上游平台自身提供的强关联证明！
3. **证据 C：短链接溯源闭环（Short URL Redirection Closure）**  
   * 旧广播中的短链接跳转或历史网页中的重定向关系形成闭环。
4. **证据 D：用户第一人称言论的上下文佐证（User Mark Semantic Evidence）**  
   * 当用户在新条目的标记评语中明确陈述：“*我很确定以前还叫《情感反诈模拟器》的时候就标记过，豆瓣删条目又重建条目了*”，系统通过命名实体与时间轴比对，将此作为辅助判定凭证。

#### 2.3.2 实体对齐产物模型 (`entities.ndjson`)
当对齐成立时，生成规范实体记录：

```jsonc
{
  "entity_id": "entity:game:revenge-on-gold-diggers",
  "primary_subject_id": "33375066", // 优先使用当前存活活跃的条目 ID
  "display_title": "捞女游戏 Revenge on Gold Diggers",
  "members": [
    {
      "medium": "game",
      "subject_id": "37364867",
      "status": "tombstone",
      "evidence": "keywords_match:情感反诈模拟器"
    },
    {
      "medium": "game",
      "subject_id": "33375066",
      "status": "active",
      "evidence": "upstream_current"
    }
  ],
  "same_as": [
    "https://www.douban.com/game/37364867/",
    "https://www.douban.com/game/33375066/",
    "https://store.steampowered.com/app/3057160/"
  ],
  "external_ids": {
    "steam": "3057160"
  },
  "alignment_rule": "external_id_and_keyword_match",
  "confidence": 0.99
}
```

---

### 2.4 模块四：元信息结构化提取 (`raw_meta` Parser)

根据 `canonical/FIELDS.md` §4，列表页提取的 `intro` / `pub` / `desc` 是未分拆的纯文本字符串。Enricher 承担这层有损但高价值的推断工作。

#### 2.4.1 跨媒介提取模式库

| 媒介 | 真实字符串示例 | 提取目标字段 | 提取判据与启发式 |
|---|---|---|---|
| **游戏 (`desc`)** | `PC / MAC / LIN / IPHN / ANDR / PS5 / XSX / NS / NS 2 / PS4 / XONE / 文字冒险 / 益智 / 模拟 / 2025-06-19` | `platforms`, `genres`, `release_dates` | 平台词典闭集匹配（PC、MAC、PS5、NS 2 等）；游戏类型库匹配；ISO 日期提取。 |
| **图书 (`pub`)** | `[美] 罗伯特·T·清崎 / 萧明 / 四川人民出版社 / 2019-8-1 / 89.00元` | `authors`, `translators`, `publisher`, `pub_date`, `price` | 正则识别末尾价格 (`\d+元|\$|￥`)；日期提取 (`\d{4}-\d{1,2}`)；国籍前缀识别 (`\[.+?\]`)；出版社名单库比对。 |
| **音乐 (`intro`)**| `星野源 / 2016-10-05 / Limited Edition / CD / 流行` | `artists`, `release_date`, `edition`, `media_format`, `genres` | 介质字典 (`CD\|Vinyl\|LP\|数字`)；日期提取；流派词典 (`流行\|摇滚\|民谣\|爵士`)。 |
| **影视 (`intro`)**| `2026-01-23(美国/中国大陆) / 杰瑞米·艾文 / ... / 103分钟 / 悬疑 / 英语` | `release_date`, `regions`, `cast`, `runtime_minutes`, `genres`, `languages` | 正则识别时长 (`\d+分钟`)；全球国家/地区词典；ISO 语言词典；首位上映日提取。 |

#### 2.4.2 交叉验证与自校验
* 如果条目本身拥有详情页捕获的 `#info`（原生携带中文标签）：
  * 提取器会将 `raw_meta` 的提取结果与 `#info` 逐项交叉比对；
  * 比对一致的项，置信度标记为 `1.0`；
  * 若条目无详情页（如列表页纯墓碑或抓取不全），提取结果标记为 `confidence: 0.85` 并注明 `source: "extracted:heuristic_v1"`。

---

### 2.5 模块五：语言与别名智能标注 (Language & Alias Tagging)

CLAUDE.md 明确定规：“*The parser must not guess a language tag. Douban's 又名 list mixes Cantonese, Taiwanese, English and transliterations untagged. Write `lang: null`. Language detection is enrichment, gets `source: 'detected'` plus a confidence, and can be re-run.*”

#### 2.5.1 零依赖轻量语言检测器
在不引入庞大 NLP 依赖的前提下，利用 Unicode 字符区间与特定词汇特征实现高精度判定：
1. **纯 ASCII / 罗马字符**：判定为 `en` 或原文语言。
2. **日文假名（平假名 `\u3040-\u309F` / 片假名 `\u30A0-\u30FF`）**：判定为 `ja`。
3. **韩文字母（`\uAC00-\uD7AF`）**：判定为 `ko`。
4. **汉字别名细分**：
   * 包含港台特有繁体字库或后缀带有 `(台)`、`(港)`、`(港/台)`：标注为 `zh-Hant`（以及对应细分 `zh-TW` / `zh-HK`）。
   * 包含常见简体规范字：标注为 `zh-Hans`。
   * 拼音转写特征（如带声调字母或空格分隔拼音）：标注为 `zh-Latn-pinyin`。

---

## 3. 便携式增强归档包标准 (Portable Enrichment Bundle Specification)

为了满足用户**“和 Bundle 一样可以备份、带走、长期冷存”**的根本诉求，Enricher 的产物不是单机临时缓存，而是一个自包含、标准化的第一公民归档包。

### 3.1 目录布局标准

```
doubak-enrichment-<enrichment_id>/
├── README.txt                               ← 双语纯文本说明（面向 2040 年，说明此归档的来龙去脉与读取方法）
├── manifest.json                            ← 归档清单（关联账号、依赖的 canonical 版本、证据段 SHA-256、条目统计）
│
├── ─── 上层语义层（供下游工具零开销秒读）
├── subjects.enriched.ndjson                 ← 增强后的作品元数据（标题、别名、语言、外部 ID）
├── entities.ndjson                          ← 实体对齐表（记录 37364867 ⟷ 33375066 聚合映射）
│
└── ─── 底层凭证层（可司法取证、可离线重放的真实网络报文）
    ├── index-<enrichment_id>.ndjson         ← 外部请求捕获索引（URL、偏移量、长度、SHA-256、Intent）
    └── data-<enrichment_id>-00001.warc.gz   ← 标准 WARC 1.1（封存所有外部网络交互的原始响应）
```

### 3.2 为什么将网络请求存为 WARC 是可备份归档的最佳选择？
1. **司法取证级可复现性（Forensic Reproducibility）**：十年之后（如 2036 年），即便 Steam 变更了 API 协议、Wayback Machine 遭受网络阻断，归档包内依然完好封存着 2026 年 Steam 官方服务器返回的原始 HTTP 响应报文与数字头部，以及 Wayback 返回的豆瓣已删除原网页 HTML 字节，具有不可辩驳的证据力。
2. **双层解耦设计**：
   * 下游静态站点生成器（`doubak-site-generator`）与导出工具（`doubak-export-adapters`）**直接读取顶层的 `subjects.enriched.ndjson` 与 `entities.ndjson`**，享受纯文本、$O(1)$ 速度与极简性；
   * 底层的 `data-*.warc.gz` 专注于冷备存证与提供图片原始字节。
3. **零代码复用现有多媒体提取管道**：
   `doubak-site-generator/src/images.js` 原生支持扫描 `index-*.ndjson` 并按偏移量从 `data-*.warc.gz` 中解压图片。通过复用这一机制，站点生成器**无需编写任何新代码**，即可把 `doubak-enrichment-*` 当作普通 bundle 统一解压封面图到 `static/covers/`！

### 3.3 模式定义 (JSON Schemas)

#### 3.3.1 `subjects.enriched.ndjson`
每行一个合法 JSON 对象：

```jsonc
{
  "enrichment_version": "enrichment/1.0",
  "medium": "game",
  "id": "37364867",
  "entity_id": "entity:game:revenge-on-gold-diggers",
  
  // 墓碑条目恢复声明
  "tombstone": {
    "is_upstream_deleted": true,
    "recovered": true,
    "recovery_tier": "wayback_and_steam"
  },

  // 恢复的标题（带证据来源）
  "title": {
    "value": "情感反诈模拟器",
    "source": "wayback:20250815T120000Z",
    "confidence": 0.95,
    "enriched_at": "2026-10-07T09:30:00Z"
  },

  // 结构化别名库（带语言与来源标注）
  "aliases": [
    {
      "value": "捞女游戏",
      "lang": "zh-Hans",
      "source": "wikidata:Q131920199",
      "confidence": 0.90
    },
    {
      "value": "Revenge on Gold Diggers",
      "lang": "en",
      "source": "steam:3057160",
      "confidence": 0.98
    }
  ],

  // 全球通用外部唯一标识
  "external_ids": {
    "wikidata": "Q131920199",
    "steam": "3057160",
    "imdb": null
  },

  // 恢复的海报（关联至包内 WARC 证据索引）
  "cover": {
    "url_key": "https://shared.fastly.steamstatic.com/store_item_assets/steam/apps/3057160/header.jpg",
    "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "source": "steam",
    "confidence": 0.98
  },

  // 从 raw_meta 结构化提取的结果
  "extracted_meta": {
    "platforms": {
      "value": ["PC"],
      "source": "extracted:heuristic_v1",
      "confidence": 0.95
    },
    "genres": {
      "value": ["文字冒险", "益智", "模拟"],
      "source": "extracted:heuristic_v1",
      "confidence": 0.90
    },
    "release_date": {
      "value": "2025-06-19",
      "source": "steam:3057160",
      "confidence": 0.98
    }
  }
}
```

---

## 4. 规范演进规划：`doubak-data-specs` 需要更新的规范清单 (Specifications Evolution)

引入便携式增强归档包后，规范仓库 `doubak-data-specs` 需要相应确立正式的技术规范：

### 4.1 新增第三棵独立的规范树：`enrichment/`
在 `doubak-data-specs` 根目录下，与 `bundle/` 和 `canonical/` 并列确立第三棵树：

| 规范目录 | 写入方 | 生命周期 | 核心使命 |
|---|---|---|---|
| `bundle/` | 抓取工具与导入器 | 用户抓取完成后即**永久冻结** | 忠实记录不可逆的抓取行为与原始字节 |
| `canonical/` | 解析器 | **随时可迭代演进** | 从 WARC 提取纯粹的客观观测事件日志 |
| **`enrichment/`** | **数据关联器** | **可按需重跑、可独立打包备份** | **建立跨平台实体对齐映射与外部知识关联** |

* **规范文件落地**：
  * `enrichment/README.md` —— 规范树入口与设计哲学；
  * `enrichment/v1/SPEC.md` —— 归档包目录布局、校验和算法、字段定义与时间戳语义；
  * JSON Schema 集合：
    * `manifest.schema.json`（定义 `spec_version: "enrichment/1.0"`）；
    * `subject-enriched.schema.json`（定义增强字段、证据来源与置信度）；
    * `entity.schema.json`（定义实体成员与 `same_as` 映射）；
  * 词表注册表：
    * `vocabularies/enrichment-source.json`（白名单枚举：`local_archive`, `wayback`, `wikidata`, `steam`, `tmdb` 等）；
    * `vocabularies/alignment-rule.json`（对齐规则枚举：`external_id_match`, `keyword_match` 等）；
  * 零依赖校验器：`enrichment/v1/validate.py <归档包路径>`。

### 4.2 `bundle/v1/` 词表的兼容性扩展
为了让 `doubak-enrichment-<id>` 内部凭证段使用的 `data-*.warc.gz` 与 `index-*.ndjson` 能够**直接通过现有 `bundle/v1/validate.py` 的校验**，需要在 `bundle/v1/vocabularies/intent.json` 中追加注册 Enricher 的意图：

* `enrichment.tombstone.wayback_cdx`：请求 Wayback CDX API 索引；
* `enrichment.tombstone.wayback_snapshot`：获取历史原网页 HTML 快照；
* `enrichment.entity.wikidata_sparql`：查询 Wikidata SPARQL 端点；
* `enrichment.entity.wikidata_item`：拉取 Wikidata Entity JSON；
* `enrichment.domain.steam_app`：拉取 Steam 官方 Storefront API；
* `enrichment.domain.tmdb_item`：拉取 TMDB 影视元数据；
* `enrichment.asset.cover`：下载恢复出的高清封面图字节。

### 4.3 `canonical/` 规范的交叉引用补充
* **`canonical/FIELDS.md` §4（`raw_meta` 存储粒度）**：
  补齐交叉引用，明确指出：“*解析器原样存储的斜杠字符串，由下游 `enrichment/v1` 的规则提取器拆解为结构化字段，存放于 `subjects.enriched.ndjson` 的 `extracted_meta` 命名空间下。*”
* **`canonical/FIELDS.md` §3（墓碑占位符判定）**：
  明确指出：“*解析器置为 `title: null` 的墓碑条目，下游 `enrichment/v1` 通过证据链挽救恢复真实标题，并在静态站点与导出适配器中安全消费。*”
* **`canonical/IDENTITY.md` §2.4（跨观测归并与删掉再重标）**：
  明确指出：“*上游删除后异 ID 重建产生的两条独立 canonical 条目，在 canonical 保持纯净的前提下，由 `enrichment/v1` 的 `entities.ndjson` 统一聚合为现实作品实体，指导下游站点与导出层的平滑合并。*”

---

## 5. 下游组件集成规范 (Downstream Contracts)

### 5.1 与 `doubak-site-generator` 的契约
* 命令行约定：
  ```sh
  npm run site -- <canonical> <bundles> [out] [--enrichment <enrichment_bundle_dir>]
  ```
* 若未传递 `--enrichment`，系统严格保持现有降级行为运行。
* `projection.js` 在加载 Enricher 产出后：
  1. `mergeReMarks` 按 `entity_id` 聚合，跨 ID 重建作品（如 `37364867` 与 `33375066`）的标记与广播自动合并；
  2. 墓碑作品的 `title` 由恢复出的有效字段填补；
  3. **封面图片零新代码提取**：直接将 `doubak-enrichment-<id>/` 传入 `images.js`，按原有逻辑提取封存的图片到 `static/covers/`，**绝对不向外网发起请求**；
  4. `markdown.js` 生成跳转别名，主页面呈现客观对齐依据。

### 5.2 与 `doubak-export-adapters` 的契约
* 命令行约定：
  ```sh
  node bin/export.js <canonical> [out] [--enrichment <enrichment_bundle_dir>]
  ```
* 导出至 NeoDB 时，若某条目在豆瓣已是墓碑，但已被 Enricher 对齐至存活的新条目或有效的 Steam/IMDb 外部页面，则使用有效链接输出，**避免被抛弃进 `neodb-needs-check.csv`**。

---

## 6. 参考架构：NeoDB 跨数据源映射与条目合并机制深度剖析 (Reference Architecture: Multi-Source Mapping in NeoDB)

为了确保设计具备工业级鲁棒性并能与联邦宇宙无缝互通，本节深入 NeoDB 核心源码（基于本地仓库 `/home/mewx/codes/neodb`），系统梳理 NeoDB 在处理多数据源映射与条目合并时的成熟实践，并将其提炼为 Doubak Enricher 的规范指导。

### 6.1 核心数据模型：`Item` 与 `ExternalResource` 的 1:N 枢纽架构
NeoDB 的核心模型定义在 `catalog/models/item.py` 与 `catalog/models/common.py` 中：

```
                ┌───────────────────────────────────────┐
                │          NeoDB Item (现实作品)        │
                │  (Game / Movie / TVSeason / Edition)  │
                └───────────────────────────────────────┘
                                    ▲
                                    │ (1 : N 外键关联)
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│ExternalResource │        │ExternalResource │        │ExternalResource │
│豆瓣旧条目 37364867│        │豆瓣新条目 33375066│        │Steam App 3057160│
│(IdType:         │        │(IdType:         │        │(IdType: Steam,  │
│ DoubanGame)     │        │ DoubanGame)     │        │ other_lookup_ids│
└─────────────────┘        └─────────────────┘        └─────────────────┘
```

* **`Item`（多态实体）**：代表现实中的作品实体（Game, Movie, Edition 等），是所有用户标记、评分与书架动态的主锚点。
* **`ExternalResource`（外部数据源快照）**：代表特定网站上的条目页面。关键字段包含：
  * `id_type`：站点分类枚举（如 `doubangame`, `doubanmovie`, `steam`, `imdb` 等）；
  * `id_value`：该站点的唯一主键（如 `33375066`, `3057160`, `tt...`）；
  * `url`：该条目的规范 URL；
  * **`other_lookup_ids`（核心资产）**：一个 JSON 字典，保存**该资源自身附带的其他平台全局唯一 ID**（例如在豆瓣电影页面上抓到的 IMDb 号，或图书页提取的 ISBN）。

### 6.2 核心对齐算法：`_match_existing_item` 的五级降级匹配链
在 `item.py:1550` 中，NeoDB 定义了严格的**五级唯一键级联匹配算法**，用于判断一个外部资源是否已经对应库中的某部作品：

```python
"""
try match an existing Item in the following order:
1. id_type/id_value 匹配 Item 的主主键 (primary_lookup_id)
2. any other_lookup_ids 匹配 Item 的主主键
3. id_type/id_value 匹配库中已有外部资源的 other_lookup_ids
4. any other_lookup_ids 匹配库中已有外部资源的 id_type/id_value
5. any other_lookup_ids 与库中已有外部资源的 other_lookup_ids 相交匹配
"""
```

**工程启示**：
* **零中文标题模糊匹配**：NeoDB 坚决不做中文名称的模糊猜测匹配，所有对齐全部建立在可计算、无歧义的硬性标识符上。
* **借力交叉标识（`other_lookup_ids`）实现跨平台自动归拢**：如果条目 A（来自 Steam，`id_value: 3057160`）已存在，当一个豆瓣条目 B 携带了 `other_lookup_ids: {'steam': '3057160'}` 进入系统时，第 4 级规则立即触发，NeoDB 自动将豆瓣资源挂载到原有的 Steam `Item` 下，自动完成跨站数据合并！

### 6.3 标识符权威层级与 `IdealIdTypes` 哲学
在 `common.py:133` 中，NeoDB 定义了公信力最高的理想主键列表 `IdealIdTypes`：

```python
IdealIdTypes = [
    IdType.ISBN,
    IdType.CUBN,
    IdType.ASIN,
    IdType.GTIN,
    IdType.ISRC,
    IdType.OCLC,
    IdType.MusicBrainz_ReleaseGroup,
    IdType.RSS,
    IdType.IMDB,
    IdType.Steam,
    IdType.Itch,
    IdType.WikiData,
    IdType.TMDB_Person,
]
```

**关键设计洞察**：
* 豆瓣的所有私有 ID（`DoubanMovie`, `DoubanBook`, `DoubanGame`）**均不在 `IdealIdTypes` 中**！
* 商业平台的私有数字 ID 具有易变性、区域性和易被删改的脆弱性；而 ISBN、IMDb、Steam、Wikidata 是全球公认、持久存在的数字公钥。
* **结论**：Doubak Enricher 的首要任务就是**将脆弱的豆瓣 ID 映射锚定到 `IdealIdTypes` 上**。

### 6.4 豆瓣特有爬虫实现与审查下架（`RESPONSE_CENSORSHIP`）识别
NeoDB 在 `catalog/sites/douban.py:85` 的 `DoubanDownloader.validate_response` 中明文定义了对豆瓣审查页面的识别：

```python
elif response.status_code == 204:
    return RESPONSE_CENSORSHIP
elif response.status_code == 200:
    content = response.content.decode("utf-8")
    if (
        content.find("<title>页面不存在</title>") != -1
        or content.find("呃... 你想访问的条目豆瓣不收录。") != -1
        or content.find("根据相关法律法规，当前条目正在等待审核。") != -1
    ):
        return RESPONSE_CENSORSHIP
```

NeoDB 明确将这些特征判定为“审查下架”，直接中断抓取。这印证了为什么当条目被豆瓣删除后，NeoDB 会完全丧失对该条目的抓取建档能力。

### 6.5 归档导入时的链接优先级调度（`_PREFERRED_SITES`）
在用户向 NeoDB 导入备份包时，导入器基类 `journal/importers/base.py:get_item_by_info_and_links` 对条目链接执行优先级排序：

```python
_PREFERRED_SITES = [
    SiteName.Fediverse,
    SiteName.RSS,
    SiteName.TMDB,
    SiteName.IMDB,
    SiteName.GoogleBooks,
    SiteName.Goodreads,
    SiteName.IGDB,
]
```

* `SiteName.Douban` 并不在优先列表中（排序权重为默认的 99）。
* 导入器按优先级遍历条目提供的全部 `links`：
  ```python
  links = [u] + [r["url"] for r in i.get("external_resources") or []]
  ```
* **解决墓碑丢条目的关键机理**：
  如果归档包中仅提供已失效的豆瓣 URL（`https://www.douban.com/game/37364867/`），NeoDB 访问返回 `RESPONSE_CENSORSHIP`，解析失败导致该条目被丢弃；
  **但只要 Doubak Enricher 在 `external_resources` 中附加上 `https://store.steampowered.com/app/3057160/` 或新重建的豆瓣链接**，NeoDB 就会命中 Steam Scraper 或新豆瓣页面，条目成功被创建并与用户的标记绑定！

### 6.6 条目物理合并语义 (`merge_to`)
在 `item.py:960` 的 `merge_to` 中，NeoDB 规范了条目合并的标准行为：
1. `self.merged_to_item = to_item`；
2. **资源重挂载**：`for res in self.external_resources.all(): res.item = to_item; res.save()`；
3. **元数据合并与去重**：`uniq(getattr(to_item, k, []) + (v or []))`；
4. **历史操作平移**：用户指向旧条目的所有 `Mark`、评论与 `ShelfLog` 历史，自动顺着指针归集到合并后的目标条目上。

### 6.7 对 `doubak-data-enricher` 的直接工程启示

| NeoDB 成熟机制 | Doubak Enricher 的吸收与规范对齐 |
|---|---|
| **1:N Hub 模型** | `entities.ndjson` 充当抽象 Hub，`douban:game:37364867` 与 `douban:game:33375066` 作为观察 Spoke 挂载其下。 |
| **`other_lookup_ids` 级联匹配** | Enricher 建立跨站标识字典（`external_ids`），不靠标题模糊匹配，靠公共唯一标识实现自动聚合。 |
| **`IdealIdTypes` 权威优先级** | 确立以 ISBN、IMDb、Steam AppID、Wikidata QID 为最高权重的证据等级。 |
| **多链接容灾导入** | 在导出适配器输出的 `catalog.ndjson` 中注入完整的 `external_resources`，彻底根除 NeoDB 导入丢墓碑的问题。 |
| **`merge_to` 引用重定向** | 投影层（`projection.js`）按聚合实体平移历史时间线，静态站生成别名跳转。 |

---

## 7. 工程实现与质量保证 (Engineering & Verification)

### 7.1 零外部依赖技术选型
* **原生 HTTP 与自动打包**：采用 Node.js 原生 `fetch()`，内置流式 WARC 写入器（纯原生 Node.js 流与 `node:zlib`，零第三方 npm 依赖）。
* **限流与防风控**：对 Internet Archive 与 Wikidata 请求施加原生令牌桶限流，请求间隔 ≥ 1.5 秒，智能退避。

### 7.2 确定性测试矩阵 (`npm test`)
必须实现以下自动化测试：
1. **`priority-queue-scheduling.test.js`**：断言包含墓碑与正常条目的全量归档优先消费 P0 队列，并在 P0 完成后触发提前固化提交（Early Commit）。
2. **`tombstone-37364867.test.js`**：针对真实的墓碑条目 `37364867`，在 Mock/Cache 环境下验证其标题成功恢复为《情感反诈模拟器》，并成功提取 Steam AppID `3057160`。
3. **`entity-alignment-33375066.test.js`**：验证 `37364867`（旧）与 `33375066`（新）基于真实捕获的关键词证据与 Steam ID 证据成功对齐，下游投影时间线完整融汇 2025 年与 2026 年两次标记。
4. **`portable-bundle-verify.test.js`**：对产出的 `doubak-enrichment-<id>` 运行完整性自检，断言 WARC gzip 记录完好、SHA-256 校验通过、且能被现有的 `bundle/v1/validate.py` 校验器成功读取。

---

## 8. 实施路线图 (Milestones & Roadmap)

| 阶段 | 交付目标 | 核心工作内容 |
|---|---|---|
| **Phase 1** | **便携归档包框架与规范制定 (Bundle Scaffolding & Spec)** | 制定 `doubak-data-specs/enrichment/v1` 规范与 JSON Schema；实现 `doubak-data-enricher` 的 WARC 写入器、优先级队列调度器（P0 丢失优先）与归档包骨架。 |
| **Phase 2** | **元信息提取器与语言标注 (RawMeta & Lang Detector)** | 实现针对游戏/电影/图书/音乐的 `raw_meta` 启发式规则提取器与零依赖 CJK 语言判定器。 |
| **Phase 3** | **Wayback 快照与外部 ID 反查 (Automated Tombstone Recovery)** | 实现 Wayback Machine CDX API 客户端与 Wikidata SPARQL 客户端；以 `37364867` 墓碑为基准跑通恢复并封存进 WARC。 |
| **Phase 4** | **垂直领域知识库对接 (Domain Knowledge Bases)** | 接入 Steam Storefront API 与 TMDB API，拉取官方海报字节写入 WARC。 |
| **Phase 5** | **下游流水线贯通 (Downstream Integration & E2E)** | 升级 `doubak-site-generator` 与 `doubak-export-adapters`，实现针对 `37364867` ⟷ `33375066` 的全链路平滑合并渲染，并在 NeoDB 导入中实测验证多链接容灾。 |

---

## 9. 结语

`doubak-data-enricher` 坚决拒绝主观的人工数据篡改，以条目丢失抢救为最高优先级使命，依靠历史档案快照、全球公共知识图谱与客观证据链，将每一次外部求证真实地封存在可便携、可冷备的归档包中。它为每一个被平台审查删除或异名重建的作品找回属于它的真实身份，守护数字时代里每一个普通人不可磨灭的文化足迹。
