# doubak-data-enricher 产品需求与架构设计说明书 (PRD)

> **文档状态：** 架构设计定稿 (Approved Architecture Baseline)  
> **面向仓库：** `doubak-data-enricher`  
> **适用版本：** `enrichment/1.0`  
> **关联规范：** `doubak-data-specs` (canonical/1.1, bundle/1.4), `doubak-site-generator`, `doubak-export-adapters`

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
                                             [enrichment/ 增强缓存层]
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
本仓库的架构与工程设计必须无条件恪守以下五条铁律：

1. **客观事实与派生缓存分离 (Immutable Facts vs. Derived Cache)**  
   WARC 原始捕获与用户标记是不可撼动的客观事实；`canonical` 是客观观测事件日志。Enricher 产出的所有外部 ID、推断元数据及恢复信息**纯属衍生缓存（Derived Cache）**。清空 Enricher 产出，整个系统依靠原始捕获依然能离线全量构建。
2. **绝不允许主观手填元数据，一切增强必须基于可信证据链 (No Manual Metadata Injection)**  
   严禁提供主观手工篡改标题、海报、年份等元数据的后门。手填元数据不仅无法保证真实性，还会引入不可维护的第二混乱真相源。Enricher 的所有数据补充，**必须来自有据可查、可重复验证的公开源**（历史抓取快照、Wayback Machine、Wikidata、Steam、TMDB、Bangumi 等）。
3. **构建与渲染阶段绝不发起网络请求 (Zero Network at Build/Render Time)**  
   Enricher 是整个流水线中**唯一**被允许发起外部网络请求的组件。一旦抓取完成，所有关联结果及网络原始响应必须**固化落盘在本地**。静态站点生成（`site-generator`）与向第三方导出（`export-adapters`）必须永远保持 100% 离线运行。
4. **下游非强制依赖 (Strictly Optional Downstream)**  
   任何下游工具绝不得强制依赖 Enricher。没有 Enricher 产出时，下游依靠纯 `canonical` 必须能无缝退化工作。
5. **极简审计性与零运行时依赖 (Zero Runtime Dependencies)**  
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
2026-10-06  用户重标 33375066 (捞女游戏) ─────────┴─> 【两份分离的记录！现实中却是同一部作品】
```

* **痛点现状**：
  1. **Canonical 层必须保持分离**：根据 `IDENTITY.md` §2.4，`37364867` 与 `33375066` 是两个不同时期、不同 ID 的客观观测，解析器绝不能将其合并成一个，否则就破坏了历史观测的真实性。
  2. **静态站点上的分裂体验**：若无 Enricher，站点会生成两个孤立页面 —— `game/37364867.html`（叫“未知作品”，带着 2025 年的旧短评）与 `game/33375066.html`（叫“捞女游戏”，带着 2026 年的吐槽），历史脉络彻底断裂。
  3. **导出适配器上的残缺**：向 NeoDB 导出时，旧标记 `37364867` 因上游 URL 为 null 只能被抛弃在 `neodb-needs-check.csv`，而新标记 `33375066` 缺乏 2025 年那段完整的历史演进。

---

## 2. 系统功能规范 (Functional Specifications)

`doubak-data-enricher` 拒绝任何形式的主观手工编造数据，所有能力均基于**确定性的机器规则、可信凭证与证据链**。系统分为四大核心模块：

```
                    ┌────────────────────────────────────────────────────────┐
                    │                 doubak-data-enricher                   │
                    └────────────────────────────────────────────────────────┘
                                                 │
         ┌───────────────────┬───────────────────┴───────────────────┬───────────────────┐
         ▼                   ▼                                       ▼                   ▼
┌─────────────────┐ ┌─────────────────┐                     ┌─────────────────┐ ┌─────────────────┐
│ 1. 墓碑证据恢复 │ │ 2. 实体证据对齐 │                     │ 3. 元信息提取器 │ │ 4. 语言标记器   │
│ (Tombstone Rec) │ │ (Entity Align)  │                     │ (RawMeta Parser)│ │ (Lang Tagging)  │
└─────────────────┘ └─────────────────┘                     └─────────────────┘ └─────────────────┘
  ├─ 历史抓取快照比对 ├─ 全球唯一标识 (Steam/IMDb)            ├─ 封闭词典比对     ├─ Unicode 字符集判定
  ├─ Wayback CDX API  ├─ 页面关键词与又名交叉验证            ├─ 格式模式匹配     ├─ 简繁粤英归类
  ├─ Wikidata SPARQL  └─ 用户上下文证据对齐                  ├─ #info 表格比对   └─ BCP 47 编码标注
  └─ 垂直数据库精确匹配                                      └─ 置信度加权打分
```

---

### 2.1 模块一：墓碑证据恢复引擎 (Tombstone Recovery Engine)

针对 `upstream_deleted === true`（`title === null`）的墓碑条目，必须通过**客观凭证链**找回真实元数据，拒绝任何“凭空捏造”：

#### 2.1.1 凭证链恢复流水线

1. **Tier 1: 本地跨版本/历史快照回溯 (Local Archive Traceback)**  
   * **原理**：检查本地历史所有 bundle 中，是否在该条目被删除前曾抓到过其详情页。  
   * **实测成果**：真实归档中，`movie/11611021`《在这世界的角落》与 `game/24299254`《瘟疫公司》即在此层 100% 恢复。  
   * **置信度**：`source: "local_history:bundle_id"`, `confidence: 1.0`。

2. **Tier 2: Wayback Machine CDX API 历史快照提取**  
   * **原理**：针对豆瓣原始 URL（如 `www.douban.com/game/37364867/`）或广播中捕获的短链（如 `douc.cc/2GLwai`），请求 Internet Archive CDX API：  
     `https://web.archive.org/cdx/search/cdx?url=www.douban.com/game/37364867/&output=json&filter=statuscode:200`
   * **解析**：下载最近一次状态正常的存档快照，重用解析器的原生抽取逻辑提取当时的 `<title>`、`#info` 与海报。  
   * **置信度**：`source: "wayback:<timestamp>"`, `confidence: 0.95`。

3. **Tier 3: Wikidata 结构化属性精准检索 (SPARQL)**  
   * **原理**：通过 Wikidata 登记的权威属性反向检索：  
     * 游戏：`wdt:P11867` (Douban Game ID)  
     * 影视：`wdt:P4438` (Douban Movie ID)  
     * 图书：`wdt:P11868` (Douban Book ID)  
   * **产出**：通过属性关联，获取官方多语言名、Steam AppID (`P1733`)、IMDb ID (`P345`)。  
   * **置信度**：`source: "wikidata:Q..."`, `confidence: 0.90`。

4. **Tier 4: 垂直领域官方数据库比对 (Steam / TMDB / Bangumi)**  
   * **原理**：当获得确定性的外部 ID（如 Steam AppID `3057160`）时，调用官方 API 拉取经过数字签名的权威官方元数据与封面。  
   * **置信度**：`source: "steam:3057160"`, `confidence: 0.98`。

---

### 2.2 模块二：基于证据的实体对齐与条目合并 (Evidence-based Entity Alignment)

针对豆瓣上游“删除后换 ID 重建”的问题，Enricher 建立独立的实体对齐层，**不修改 canonical 历史，而是在上层产出聚合映射 (`entities.ndjson`)**。

#### 2.2.1 自动对齐的四大证据准则 (Zero-Guesswork Alignment Rules)
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

#### 2.2.2 实体对齐产物模型 (`entities.ndjson`)
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

#### 2.2.3 下游合并消费协议

##### A. 静态站点生成器 (`doubak-site-generator`) 如何呈现合并
* **投影合并聚合 (`projection.js`)**：
  * 原本的 [`mergeReMarks`](file:///home/mewx/codes/doubak/doubak-site-generator/src/projection.js#L54-L96) 仅在同一 `(medium, subject.id)` 内部合并（处理“同一条目删标重标”）。
  * 引入 Enricher 后，`mergeReMarks` 接入 `entity_id` 聚类。
  * **效果**：旧条目 `37364867` 的 2025 年短评、星级、广播时间线，与新条目 `33375066` 的 2026 年最新标记**完整融合成一条完整作品时间线**！
* **页面生成与重定向 (`markdown.js`)**：
  * 仅以新 ID 渲染单一作品主页（`game/33375066.md`）。
  * 旧墓碑 URL `game/37364867.html` 自动生成别名跳转（利用 Hugo `aliases: ["/game/37364867/"]`）。
  * 页面展示由证据驱动的提示徽章：
    > ℹ️ *条目合并存证：本作品曾以 ID 37364867 收录，下架后于 33375066 重建。两处记录的时间线与短评已基于 Steam AppID 3057160 与条目别名凭据自动合并。*

##### B. 导出适配器 (`doubak-export-adapters`) 如何处理合并
* 导出至 NeoDB 时：
  * 原先旧条目 `37364867` 因上游 URL 为 null 会被当作无法识别丢弃；
  * 合并后，旧标记的 2025 年历史作为 `ShelfLog` 挂载到主实体下；
  * `catalog.ndjson` 优先写入有效的 Steam 外部链接或新条目链接 `https://www.douban.com/game/33375066/`；
  * **成果**：旧标记不再丢失，成功完整导入 NeoDB！

---

### 2.3 模块三：元信息结构化提取 (`raw_meta` Parser)

根据 `canonical/FIELDS.md` §4，列表页提取的 `intro` / `pub` / `desc` 是未分拆的纯文本字符串。Enricher 承担这层有损但高价值的推断工作。

#### 2.3.1 跨媒介提取模式库

| 媒介 | 真实字符串示例 | 提取目标字段 | 提取判据与启发式 |
|---|---|---|---|
| **游戏 (`desc`)** | `PC / MAC / LIN / IPHN / ANDR / PS5 / XSX / NS / NS 2 / PS4 / XONE / 文字冒险 / 益智 / 模拟 / 2025-06-19` | `platforms`, `genres`, `release_dates` | 平台词典闭集匹配（PC、MAC、PS5、NS 2 等）；游戏类型库匹配；ISO 日期提取。 |
| **图书 (`pub`)** | `[美] 罗伯特·T·清崎 / 萧明 / 四川人民出版社 / 2019-8-1 / 89.00元` | `authors`, `translators`, `publisher`, `pub_date`, `price` | 正则识别末尾价格 (`\d+元|\$|￥`)；日期提取 (`\d{4}-\d{1,2}`)；国籍前缀识别 (`\[.+?\]`)；出版社名单库比对。 |
| **音乐 (`intro`)**| `星野源 / 2016-10-05 / Limited Edition / CD / 流行` | `artists`, `release_date`, `edition`, `media_format`, `genres` | 介质字典 (`CD\|Vinyl\|LP\|数字`)；日期提取；流派词典 (`流行\|摇滚\|民谣\|爵士`)。 |
| **影视 (`intro`)**| `2026-01-23(美国/中国大陆) / 杰瑞米·艾文 / ... / 103分钟 / 悬疑 / 英语` | `release_date`, `regions`, `cast`, `runtime_minutes`, `genres`, `languages` | 正则识别时长 (`\d+分钟`)；全球国家/地区词典；ISO 语言词典；首位上映日提取。 |

#### 2.3.2 交叉验证与自校验
* 如果条目本身拥有详情页捕获的 `#info`（原生携带中文标签）：
  * 提取器会将 `raw_meta` 的提取结果与 `#info` 逐项交叉比对；
  * 比对一致的项，置信度标记为 `1.0`；
  * 若条目无详情页（如列表页纯墓碑或抓取不全），提取结果标记为 `confidence: 0.85` 并注明 `source: "extracted:heuristic_v1"`。

---

### 2.4 模块四：语言与别名智能标注 (Language & Alias Tagging)

CLAUDE.md 明确定规：“*The parser must not guess a language tag. Douban's 又名 list mixes Cantonese, Taiwanese, English and transliterations untagged. Write `lang: null`. Language detection is enrichment, gets `source: 'detected'` plus a confidence, and can be re-run.*”

#### 2.4.1 零依赖轻量语言检测器
在不引入庞大 NLP 依赖的前提下，利用 Unicode 字符区间与特定词汇特征实现高精度判定：
1. **纯 ASCII / 罗马字符**：判定为 `en` 或原文语言。
2. **日文假名（平假名 `\u3040-\u309F` / 片假名 `\u30A0-\u30FF`）**：判定为 `ja`。
3. **韩文字母（`\uAC00-\uD7AF`）**：判定为 `ko`。
4. **汉字别名细分**：
   * 包含港台特有繁体字库或后缀带有 `(台)`、`(港)`、`(港/台)`：标注为 `zh-Hant`（以及对应细分 `zh-TW` / `zh-HK`）。
   * 包含常见简体规范字：标注为 `zh-Hans`。
   * 拼音转写特征（如带声调字母或空格分隔拼音）：标注为 `zh-Latn-pinyin`。

#### 2.4.2 结构化输出
为每个别名包装元数据对象：
```json
{
  "title": "Revenge on Gold Diggers",
  "lang": "en",
  "type": "official_english",
  "source": "detected:script_v1",
  "confidence": 0.98
}
```

---

## 3. 数据规格与本地存储设计 (Data Specification & Storage Design)

### 3.1 目录布局标准
关联器执行后的本地产出严格与原始数据保持隔离，采用类似 canonical 的人类可读 NDJSON 格式存储：

```
~/downloads/enrichment/
├── README.txt                       ← 双语档案说明文件（面向 2040 年阅读者）
├── manifest.json                    ← 本次关联运行摘要与统计
├── subjects.enriched.ndjson         ← 增强后的作品元数据（一一映射或补充 canonical）
├── entities.ndjson                  ← 实体对齐表（记录 evidence_based same_as 映射）
└── .cache/                          ← 外部响应原始内容离线缓存（保证 100% 可复现与断网重跑）
    ├── wayback/                     ← Wayback Machine 原始 HTML/JSON 快照响应
    ├── wikidata/                    ← Wikidata SPARQL JSON 结果缓存
    └── steam/                       ← Steam API 原始响应
```

### 3.2 模式定义 (JSON Schemas)

#### 3.2.1 `subjects.enriched.ndjson`
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

  // 恢复的海报（本地离线相对路径）
  "cover": {
    "local_path": "covers/game_37364867.jpg",
    "remote_url": "https://shared.fastly.steamstatic.com/store_item_assets/steam/apps/3057160/header.jpg",
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

## 4. 下游组件集成规范 (Downstream Contracts)

### 4.1 与 `doubak-site-generator` 的契约
* 命令行约定：`npm run site -- <canonical> <bundles> [out] [--enrichment <dir>]`
* 若未传递 `--enrichment`，系统严格保持现有降级行为运行。
* `projection.js` 在加载 Enricher 产出后：
  1. `mergeReMarks` 按 `entity_id` 聚合，跨 ID 重建作品的标记与广播自动合并；
  2. 墓碑作品的 `title` 与 `coverUrl` 由恢复出的有效字段填补，封面引用本地离线图片，**绝对不向外网发起请求**；
  3. `markdown.js` 生成跳转别名，主页面呈现客观对齐依据。

### 4.2 与 `doubak-export-adapters` 的契约
* 命令行约定：`node bin/export.js <canonical> [out] [--enrichment <dir>]`
* 导出至 NeoDB 时，若某条目在豆瓣已是墓碑，但已被 Enricher 对齐至存活的新条目或有效的 Steam/IMDb 外部页面，则使用有效链接输出，**避免被抛弃进 `neodb-needs-check.csv`**。

---

## 5. 工程实现与质量保证 (Engineering & Verification)

### 5.1 零外部依赖技术选型
* **原生 HTTP 与自动缓存**：采用 Node.js 原生 `fetch()`，内置 `RequestCache` 模块。每个网络请求必须将完整响应落盘在 `.cache/`，确保二次执行与断网测试 100% 确定性。
* **限流与防风控**：对 Internet Archive 与 Wikidata 请求施加原生令牌桶限流，请求间隔 ≥ 1.5 秒，智能退避。

### 5.2 确定性测试矩阵 (`npm test`)
必须实现以下自动化测试：
1. **`tombstone-37364867.test.js`**：针对真实的墓碑条目 `37364867`，在 Mock/Cache 环境下验证其标题成功恢复为《情感反诈模拟器》，并成功提取 Steam AppID `3057160`。
2. **`entity-alignment-33375066.test.js`**：验证 `37364867`（旧）与 `33375066`（新）基于真实捕获的关键词证据与 Steam ID 证据成功对齐，下游投影时间线完整融汇 2025 年与 2026 年两次标记。
3. **`zero-network-assertion.test.js`**：在无网络连接状态下，断言 Enricher 能够纯依靠本地缓存 100% 成功生成一致的 NDJSON。

---

## 6. 实施路线图 (Milestones & Roadmap)

| 阶段 | 交付目标 | 核心工作内容 |
|---|---|---|
| **Phase 1** | **基础框架与实体对齐 (Entity Alignment Core)** | 搭建 `doubak-data-enricher` 基础框架、CLI 入口、基于证据链的 `entities.ndjson` 聚类模型。 |
| **Phase 2** | **元信息提取器与语言标注 (RawMeta & Lang Detector)** | 实现针对游戏/电影/图书/音乐的 `raw_meta` 启发式规则提取器与零依赖 CJK 语言判定器。 |
| **Phase 3** | **Wayback 快照与外部 ID 反查 (Automated Tombstone Recovery)** | 实现 Wayback Machine CDX API 客户端与 Wikidata SPARQL 客户端；以 `37364867` 墓碑为基准跑通恢复。 |
| **Phase 4** | **垂直领域知识库对接 (Domain Knowledge Bases)** | 接入 Steam Storefront API 与 TMDB API，本地固化海报字节。 |
| **Phase 5** | **下游流水线贯通 (Downstream Integration & E2E)** | 升级 `doubak-site-generator` 与 `doubak-export-adapters`，实现针对 `37364867` ⟷ `33375066` 的全链路平滑合并渲染。 |

---

## 7. 结语

`doubak-data-enricher` 坚决拒绝主观的人工数据篡改，而是依靠历史档案快照、全球公共知识图谱与客观证据链，为每一个被平台审查删除或异名重建的作品找回属于它的真实身份，守护数字时代里每一个普通人不可磨灭的文化足迹。
