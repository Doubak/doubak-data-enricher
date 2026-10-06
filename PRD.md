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
[doubak-data-parser] ──(纯函数、离线、客观观测)──> [canonical/ 标准归档]
                                                         │
                                                         ▼
                                             [doubak-data-enricher] <──(外部公开数据源 / 本地覆盖)
                                             (可选、带置信度、可离线重跑)
                                                         │
                                                         ▼
                                             [enrichment/ 增强缓存层]
                                                         │
                                ┌────────────────────────┴────────────────────────┐
                                ▼                                                 ▼
                    [doubak-site-generator]                           [doubak-export-adapters]
               (投影缓存 → Markdown → 独立网站)                     (无缝迁移至 NeoDB / Letterboxd / Goodreads)
```

### 0.2 为什么必须设计独立的 Enricher
豆瓣作为一个中心化平台，其目录数据存在天然的脆弱性：
1. **上游条目被彻底抹除（Tombstone / 墓碑条目）**：政治审查、商业纠纷或版权到期会导致条目从豆瓣库中彻底下架。用户个人标记虽在，但作品标题沦为「未知游戏/未知电影」、封面变成默认占位图、详情页 404。
2. **上游删除后异 ID 重建（Entity Drift & Recreation）**：条目被删除一段时间后，网友或官方可能以全新 ID 重新提交收录。此时同一现实作品在豆瓣历史上存在两个不相交的 ID，导致历史标记与新条目割裂。
3. **元数据字符串混杂未分拆（Opaque `raw_meta`）**：依据 `FIELDS.md` §4，列表页提取的 `intro` / `pub` / `desc` 是未打标签的斜杠分隔串（电影段数多达 43 种）。解析阶段严禁猜测，拆解必须移交 Enricher 标注处理。
4. **缺乏全球实体对齐与语言标注**：豆瓣的「又名」未标注语言（简/繁/粤/英/日/韩）；缺乏国际公认唯一标识（Wikidata QID、IMDb tt、Steam AppID、ISBN、MusicBrainz MBID），限制了向分布式社交网络与第三方平台的迁移质量。

### 0.3 核心设计铁律 (Architectural Invariants)
本仓库的架构与工程设计必须无条件恪守以下五条铁律：

1. **圣神数据与缓存分离 (Sacred vs. Cache)**  
   WARC 原始捕获与用户标记是不可撼动的客观事实；`canonical` 是客观观测事件日志。Enricher 产出的所有外部 ID、推断元数据及恢复信息**纯属衍生缓存（Derived Cache）**。清空 Enricher 产出，整个系统依靠原始捕获依然能离线全量构建。
2. **构建与渲染阶段绝不发起网络请求 (Zero Network at Build/Render Time)**  
   Enricher 是整个流水线中**唯一**被允许发起外部网络请求的组件。一旦抓取完成，所有关联结果及网络原始响应必须**固化落盘在本地**。静态站点生成（`site-generator`）与向第三方导出（`export-adapters`）必须永远保持 100% 离线运行。
3. **下游非强制依赖 (Strictly Optional Downstream)**  
   任何下游工具绝不得强制依赖 Enricher。没有 Enricher 产出时，下游依靠纯 `canonical` 必须能无缝退化工作。
4. **客观观测与主观推断界限分明 (Facts vs. Inferences)**  
   解析器记录的是「那一刻豆瓣页面如实说了什么」；Enricher 记录的是「我们通过什么规则推断了什么」。Enricher 输出的每一项增强字段，必须显式携带 `source`（来源标识）、`confidence`（置信度 0.0 ~ 1.0）与 `enriched_at` 时间戳。
5. **极简审计性与零运行时依赖 (Zero Runtime Dependencies)**  
   遵循 Doubak monorepo 统一工具链：Node ≥ 20，纯原生 ES 模块（ESM），JSDoc 类型标注，`node:test` 单测框架，零第三方 npm 运行时依赖，零构建步骤。

---

## 1. 核心问题剖析与基准案例 (Problem Statement & Benchmark Cases)

### 1.1 贯穿全篇的基准案例：`game/37364867`《情感反诈模拟器》

为了确保设计的每一步都立足于真实数据，全篇以真实归档中的 **`game/37364867`** 作为基准分析案例：

* **作品背景**：互动叙事电影游戏《情感反诈模拟器》（Steam 英文名：*Revenge on Gold Diggers*，网友戏称《捞女游戏》）。
* **下架事件**：2026 年 2 月底，该游戏在豆瓣因题材争议被官方无预警全网删除（包括主条目与讨论区）。
* **Canonical 真实观测状态**：
  * 在 `subjects.ndjson` 中：
    ```json
    {
      "canonical_version": "canonical/1.1",
      "medium": "game",
      "id": "37364867",
      "url": null,
      "upstream_deleted": true,
      "revisions": [{
        "fields": {
          "title": null,
          "aliases": null,
          "info": null,
          "cover_url": "https://asset.doubanio.com/cuphead/ilmen-static/pics/subject/game_normal.png",
          "cover_url_key": "https://asset.doubanio.com/cuphead/ilmen-static/pics/subject/game_normal.png",
          "raw_meta": null
        }
      }]
    }
    ```
  * 在 `marks.ndjson` 中，用户的标记完整保留：
    * `marked_at`: `2025-07-19`
    * `status`: `done`（玩过）
    * `rating`: 4 星（见广播）
    * `tags`: `["中国", "游戏", "文字冒险", "益智", "互动电影"]`
    * `comment`: *“游戏做的很用心，还有教学档案，其实也很适合女生玩。这游戏要是10多年前出来就好了，那时候身边好多男PUA特别会撩妹，其实也就是这些技巧。”*
  * 在 `broadcasts.ndjson` 中，两条广播精准冻结了历史瞬间：
    * `6413189684`（2025-07-02 11:29:41）：“想玩”
    * `6532642615`（2025-07-19 19:18:39）：“玩过”，带 4 星评分及上述短评
  * 短链溯源：捕获的广播 HTML 中包含短链接 `https://douc.cc/2GLwai`，HTTP 302 重定向至 `http://www.douban.com/game/37364867/`。
* **痛点现状**：
  * **在静态站点上**：标题显示为占位符「未知作品」，封面为空，虽然展示了用户的短评与时间线，但浏览者无法得知这部作品究竟叫什么。
  * **在向 NeoDB 导出时**：`export-adapters` 检查发现 `url` 为 `null`，被无情剔除并计入 `neodb-needs-check.csv`（“连豆瓣链接都没有，没有放进 zip”）。用户用心写的长评和标记无法迁移至新家。

### 1.2 墓碑条目整体分布现状
在当前用户的全量真实归档（2950 条标记）中，共存在 **8 个上游已删除作品**：
* 电影 1 部：`11611021`（幸运的是，历史老版本捕获在删除前抓到了详情页，成功保留《在这世界的角落》标题与信息）。
* 游戏 7 部：`24299254`、`27054820`、`37364867`、`35404095`、`26794548`、`11504723`、`10734267`。
  * 其中部分早期游戏（如 `24299254`《瘟疫公司》）在老档案中有名字；
  * 但诸如 `37364867` 以及 `26794548`（标签为 `['恐怖', '台湾', '解谜', '历史']`，实为《返校》），在任何一次捕获中都未留存详情页，在 `canonical` 中标题彻底为 `null`。

### 1.3 豆瓣条目异 ID 重建（Entity Drift & Recreation）
用户核心洞察：“**don't rely Douban data because they deleted the subject entry and recreate under a different ID!**”

* **机制**：某些作品下架后，过了数月热度消退或换了发行方，豆瓣网友重新申请创建，系统为其分配了全新的数字 ID（例如假设为 `37999888`）。
* **分裂困境**：
  * 旧 ID `37364867`：持有用户 2025 年写下的真实短评、时间线、星级，但条目本身是死的；
  * 新 ID `37999888`：条目元数据活络完整，但与用户的历史没有任何关联。
* **规则冲突**：根据 `IDENTITY.md` §2.4，两个不同的上游 ID 在 `canonical` 层**绝对不能合并**（那是忠实的客观观测）。因此，**实体对齐（Entity Alignment）的职责必须且只能由 Enricher 承担**！

---

## 2. 系统功能架构与核心模块 (Functional Specifications)

`doubak-data-enricher` 由四大核心子系统组成：

```
                    ┌────────────────────────────────────────────────────────┐
                    │                 doubak-data-enricher                   │
                    └────────────────────────────────────────────────────────┘
                                                 │
         ┌───────────────────┬───────────────────┴───────────────────┬───────────────────┐
         ▼                   ▼                                       ▼                   ▼
┌─────────────────┐ ┌─────────────────┐                     ┌─────────────────┐ ┌─────────────────┐
│ 1. 墓碑挽救引擎 │ │ 2. 实体对齐中心 │                     │ 3. 元信息提取器 │ │ 4. 语言标记器   │
│ (Tombstone Rec) │ │ (Entity Align)  │                     │ (RawMeta Parser)│ │ (Lang Tagging)  │
└─────────────────┘ └─────────────────┘                     └─────────────────┘ └─────────────────┘
  ├─ Wayback CDX      ├─ Wikidata 桥接 (QID)                  ├─ 正则模式库       ├─ CJK 字符集判定
  ├─ Wikidata SPARQL  ├─ Steam/IMDb 同构映射                  ├─ 封闭词典比对     ├─ 简繁粤英归类
  ├─ Steam/Bangumi    ├─ 人工 same_as 映射                    ├─ #info 交叉验证   └─ BCP 47 编码标注
  └─ 本地快照回溯     └─ 别名归并与重定向生成                 └─ 置信度加权评分
```

---

### 2.1 模块一：墓碑挽救引擎 (Tombstone Recovery Engine)

针对 `upstream_deleted === true`（或 `title === null`）的条目，启动分层挽救流水线：

```
                     [检测到墓碑条目 (title == null)]
                                    │
                                    ▼
                     [Tier 0: 本地人工覆盖 overrides.yaml]
                        ├── 命中 ──> [直接采纳，置信度 1.0，终止]
                        └── 未命中
                                    │
                                    ▼
                     [Tier 1: 本地跨版本/历史快照回溯]
                        ├── 命中 ──> [采纳历史首条非空修订，置信度 1.0]
                        └── 未命中
                                    │
                                    ▼
                     [Tier 2: Wayback Machine CDX API 检索]
                        ├── 命中 ──> [抓取快照并解析，置信度 0.95]
                        └── 未命中
                                    │
                                    ▼
                     [Tier 3: Wikidata 属性反查 (SPARQL)]
                        ├── 命中 ──> [通过 P11867/P4438 提取，置信度 0.90]
                        └── 未命中
                                    │
                                    ▼
                     [Tier 4: 垂直领域数据库精准匹配 (Steam/Bangumi/TMDB)]
                        ├── 命中 ──> [外部知识库确认，置信度 0.85]
                        └── 未命中 ──> [保持 null，输出警告清单]
```

#### 2.1.1 检索策略实现细节

1. **Wayback Machine CDX API 查询**
   * **请求构造**：
     * 主 URL：`https://web.archive.org/cdx/search/cdx?url=www.douban.com/game/37364867/&output=json&filter=statuscode:200`
     * 针对影视：`movie.douban.com/subject/<id>/`
     * 广播短链接：针对捕获记录中存在的短链 `douc.cc/2GLwai` 发起检索，跟随历史 302 记录。
   * **快照提取**：若存在快照，下载最近一次成功的 WARC 或 HTML，利用 `doubak-data-parser` 的原生选择器提取 `<title>`、`#info` 与海报地址。
   * **标注**：`source: "wayback:20250815T120000Z"`, `confidence: 0.95`。

2. **Wikidata SPARQL 查询**
   * 针对不同媒介，检索对应的豆瓣标识属性：
     * 游戏：`wdt:P11867` (Douban Game ID)
     * 电影/剧集：`wdt:P4438` (Douban Movie ID)
     * 图书：`wdt:P11868` (Douban Book ID)
   * 查询模板：
     ```sparql
     SELECT ?item ?itemLabel ?steamApp ?imdbId ?pubDate WHERE {
       ?item wdt:P11867 "37364867" .
       OPTIONAL { ?item wdt:P1733 ?steamApp . }
       OPTIONAL { ?item wdt:P345 ?imdbId . }
       OPTIONAL { ?item wdt:P577 ?pubDate . }
       SERVICE wikibase:label { bd:serviceParam wikibase:language "zh,en,zh-hant". }
     }
     ```
   * 产出：获取标准中文名、外文名、Steam AppID、IMDb ID。
   * 标注：`source: "wikidata:Q..."`, `confidence: 0.90`。

3. **垂直领域数据源适配 (Steam Store API / Bangumi API)**
   * 当通过 Wikidata 拿到 Steam AppID（如 `3057160`）或通过用户评论中的线索匹配到游戏时，直接调用 Steam 官方 Storefront API：
     `https://store.steampowered.com/api/appdetails?appids=3057160&l=schinese`
   * 提取字段：`name` (情感反诈模拟器), `detailed_description`, `header_image` (官方高清海报字节), `publishers`, `genres`。
   * 标注：`source: "steam:3057160"`, `confidence: 0.98`。

---

### 2.2 模块二：实体对齐与条目合并 (Subject Entity Alignment & Merging)

解决“豆瓣删除条目后又以新 ID 重建”导致的历史分裂问题。

#### 2.2.1 实体抽象模型 (The Entity Concept)
引入 `Entity`（实体）抽象，作为高于单一平台 ID 的聚合单元：

* **实体定义**：一个现实生活中的作品（例如《情感反诈模拟器》游戏本身），拥有全局唯一的实体标识 `entity_id`（格式：`entity:<medium>:<slug_or_uuid>`）。
* **成员集合 (`members`)**：
  * 一个实体可容纳多个上游观测：
    ```json
    {
      "entity_id": "entity:game:revenge-on-gold-diggers",
      "canonical_title": "情感反诈模拟器",
      "primary_subject": { "medium": "game", "id": "37599999" },
      "members": [
        { "source": "douban", "medium": "game", "id": "37364867", "role": "tombstone" },
        { "source": "douban", "medium": "game", "id": "37599999", "role": "active" },
        { "source": "steam", "medium": "game", "id": "3057160", "role": "external" }
      ],
      "same_as": [
        "https://www.douban.com/game/37364867/",
        "https://www.douban.com/game/37599999/",
        "https://store.steampowered.com/app/3057160/"
      ]
    }
    ```

#### 2.2.2 对齐启发式与人工干预 (Alignment Engine)
1. **自动对齐判据**：
   * **外部强唯一键重合**：两个豆瓣条目若通过 Wikidata / 详情页反查拥有相同的 Steam AppID、ISBN 或 IMDb ID，则自动判定为同一实体。
   * **标题 + 核心发行时间 + 核心创作者高度重合**。
2. **人工声明优先 (`overrides.yaml`)**：
   在任何不确定的场景下，用户拥有最终决定权。Enricher 读取本地 `overrides.yaml`：
   ```yaml
   entities:
     - entity_id: "entity:game:37364867"
       title: "情感反诈模拟器"
       primary_id: "37599999"    # 若豆瓣已重建新条目，指定为主显示 ID
       merge_subjects:
         - "game:37364867"       # 旧墓碑条目
         - "game:37599999"       # 新重建条目
       external_ids:
         steam: "3057160"
         wikidata: "Q131920199"
   ```

#### 2.2.3 下游合并消费协议

##### A. 静态站点生成器 (`doubak-site-generator`) 如何呈现合并
* **投影合并升级 (`projection.js`)**：
  * 原有逻辑仅对单一 `(medium, subject.id)` 执行 `mergeReMarks`；
  * 引入 Enricher 后，`mergeReMarks` 改为依据 **`entity_id`** 聚类。
  * 聚合所有成员（无论来自旧 ID `37364867` 还是新 ID `37599999`）的历史标记与广播，生成单一完整的「说过什么」时间线！
* **页面生成与重定向 (`markdown.js`)**：
  * 主页面以 `primary_id`（如 `game/37599999.md`）渲染。
  * 若用户访问旧墓碑地址 `game/37364867.html`，Hugo 骨架自动生成 `<meta http-equiv="refresh">` 别名跳转（利用 front matter `aliases: ["/game/37364867/"]`）。
  * 页面展示提示框：
    > **条目重整说明**：此作品在豆瓣原 ID 为 `37364867`（已下架），后于 `37599999` 重建。本站已自动聚合两处记录的历史时间线与短评。

##### B. 导出适配器 (`doubak-export-adapters`) 如何处理合并
* 导出至 NeoDB 时：
  * 抛弃已失效无法访问的墓碑链接 `https://www.douban.com/game/37364867/`；
  * 优先使用外部标准链接 `https://store.steampowered.com/app/3057160/`，或重建后的活跃链接 `https://www.douban.com/game/37599999/` 注入 `catalog.ndjson` 中的 `links` 列；
  * **成果**：彻底修复当前导出中 `⚠ 5 条连豆瓣链接都没有，没有放进 zip` 的缺陷，实现 100% 成功入库。

---

### 2.3 模块三：元信息结构化提取 (`raw_meta` Parser)

根据 `canonical/FIELDS.md` §4，列表页提取的 `intro` / `pub` / `desc` 是未分拆的纯文本字符串。Enricher 承担这层有损但高价值的推断工作。

#### 2.3.1 跨媒介提取模式库

| 媒介 | 原始字符串示例 | 提取目标字段 | 提取判据与启发式 |
|---|---|---|---|
| **图书 (`pub`)** | `[美] 罗伯特·T·清崎 / 萧明 / 四川人民出版社 / 2019-8-1 / 89.00元` | `authors`, `translators`, `publisher`, `pub_date`, `price` | 正则识别末尾价格 (`\d+元|\$|￥`)；日期提取 (`\d{4}-\d{1,2}`)；国籍前缀识别 (`\[.+?\]`)；出版社名单库比对。 |
| **音乐 (`intro`)**| `星野源 / 2016-10-05 / Limited Edition / CD / 流行` | `artists`, `release_date`, `edition`, `media_format`, `genres` | 介质字典 (`CD\|Vinyl\|LP\|数字`)；日期提取；流派词典 (`流行\|摇滚\|民谣\|爵士`)。 |
| **游戏 (`desc`)** | `PC / PS5 / NS / PS4 / 角色扮演 / 2023-03-24 / 2023-03-24` | `platforms`, `genres`, `release_dates` | 平台闭集匹配 (`PC\|PS5\|PS4\|NS\|Xbox\|Switch\|Mac\|iOS\|Android`)；游戏类型库匹配。 |
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
  "title": "捞女游戏",
  "lang": "zh-Hans",
  "region": "CN",
  "type": "colloquial_alias",
  "source": "detected:script_v1",
  "confidence": 0.95
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
├── entities.ndjson                  ← 实体对齐表（记录 same_as 与多 ID 聚合关系）
├── overrides.yaml                   ← 用户本地手工标注侧车文件（可提交至私有仓库）
└── .cache/                          ← 外部响应原始内容离线缓存
    ├── wayback/                     ← Wayback Machine 原始 HTML/JSON 快照响应
    ├── wikidata/                    ← Wikidata SPARQL JSON 结果缓存
    └── steam/                       ← Steam API 原始响应
```

### 3.2 模式定义 (JSON Schemas)

#### 3.2.1 `subjects.enriched.ndjson`
每行一个合法 JSON 对象，键名与 canonical 保持一致性与可预测性：

```jsonc
{
  "enrichment_version": "enrichment/1.0",
  "medium": "game",
  "id": "37364867",
  "entity_id": "entity:game:37364867",
  
  // 墓碑条目恢复声明
  "tombstone": {
    "is_upstream_deleted": true,
    "recovered": true,
    "recovery_tier": "wayback_and_steam"
  },

  // 恢复或修正后的标题
  "title": {
    "value": "情感反诈模拟器",
    "source": "wayback:2025-08",
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
    "imdb": null,
    "bgm": null
  },

  // 恢复的海报图片（存储本地相对路径）
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
      "value": ["角色扮演", "文字冒险", "互动电影"],
      "source": "extracted:heuristic_v1",
      "confidence": 0.90
    },
    "release_date": {
      "value": "2024-08-01",
      "source": "steam:3057160",
      "confidence": 0.98
    }
  }
}
```

#### 3.2.2 `entities.ndjson`
每行记录一个跨平台/跨 ID 的聚合实体：

```jsonc
{
  "enrichment_version": "enrichment/1.0",
  "entity_id": "entity:game:37364867",
  "primary_subject_id": "37364867", // 或重建后的新 ID
  "display_title": "情感反诈模拟器",
  "members": [
    {
      "medium": "game",
      "subject_id": "37364867",
      "status": "tombstone",
      "reason": "upstream_deleted"
    },
    {
      "medium": "game",
      "subject_id": "37599999",
      "status": "active",
      "reason": "upstream_recreated"
    }
  ],
  "same_as": [
    "douban:game:37364867",
    "douban:game:37599999",
    "steam:3057160",
    "wikidata:Q131920199"
  ],
  "aligned_by": "manual", // manual | wikidata_exact | meta_heuristic
  "updated_at": "2026-10-07T09:30:00Z"
}
```

---

## 4. 下游组件集成规范 (Downstream Contracts)

### 4.1 与 `doubak-site-generator` 的契约与行为规范

#### 命令行调用约定
```sh
npm run site -- <canonical 目录> <bundle 目录> [产出目录] [--enrichment <enrichment 目录>]
```
若未传递 `--enrichment`，系统保持现有降级行为运行。

#### `projection.js` 适配修改
1. **加载 Enricher 数据**：
   在 `project()` 函数入口处，若传入 `enrichment` 选项，读取 `subjects.enriched.ndjson` 与 `entities.ndjson`，构建查找索引：
   * `entityBySubject`: `Map<"medium:id", Entity>`
   * `enrichedSubject`: `Map<"medium:id", EnrichedSubject>`
2. **`mergeReMarks` 聚类升级**：
   ```javascript
   // 原逻辑：const key = `${m.medium}:${m.subject.id}`;
   // 升级为：
   const entity = entityBySubject.get(`${m.medium}:${m.subject.id}`);
   const key = entity ? entity.entity_id : `${m.medium}:${m.subject.id}`;
   ```
   **效果**：旧墓碑 ID 与新重建 ID 的标记与广播将自动聚合进同一个作品组中！
3. **墓碑标题与海报填补**：
   在 `projectMark()` 中：
   ```javascript
   const enriched = enrichedSubject.get(`${m.medium}:${m.subject.id}`);
   const title = s?.fields?.title ?? enriched?.title?.value ?? null;
   const coverUrl = realCover(s?.fields?.cover_url) ?? enriched?.cover?.local_path ?? null;
   ```
   **离线保障**：若 Enricher 在抓取时已将恢复的海报图片持久化下载至 `covers/` 目录，则此处 `coverUrl` 为合法的本地相对路径，**绝对不向 doubanio 或第三方服务发送网络请求**，完美维护站点离线浏览原则。

#### `markdown.js` 页面渲染增强
* Front matter 注入拓展属性：
  ```yaml
  douban_upstream_deleted: true
  douban_recovered_title: true
  douban_recovery_source: "Wayback Machine / Steam"
  douban_entity_id: "entity:game:37364867"
  douban_external_links:
    steam: "https://store.steampowered.com/app/3057160/"
    wikidata: "https://www.wikidata.org/wiki/Q131920199"
  ```
* 页面模板展示：
  若检测到 `douban_recovered_title`，在标题下方呈现友好的温和徽章：
  > 📌 *本条目在豆瓣已下架，作品名称与封面系由豆备通过历史快照与公开数据库对齐恢复。*

---

### 4.2 与 `doubak-export-adapters` 的契约与行为规范

#### 命令行调用约定
```sh
node bin/export.js <canonical 目录> [输出目录] [--enrichment <enrichment 目录>]
```

#### 导出逻辑升级
1. **拯救被排除的墓碑条目**：
   * 在处理 NeoDB 导出时，目前凡是 `url === null`（即上游已删除）的条目均会被直接丢弃至 `neodb-needs-check.csv`。
   * 读取 Enricher 数据后：若作品拥有 `external_ids.steam` 或关联的重建豆瓣条目，生成规范链接：
     * `https://store.steampowered.com/app/3057160/`
     * 或关联的新豆瓣 URL
   * 该条目立即重新具备合格的匹配基准，成功进入 `catalog.ndjson`，**不再丢失任何一条标记！**
2. **外部精准标识赋能**：
   * **Letterboxd**：直接读取 `enriched.external_ids.imdb`，解决 34 部缺失 IMDb 编号的电影无法导出的问题。
   * **Goodreads**：优先读取提取的 ISBN 标识，极大提升图书匹配率。

---

## 5. 工程实现与质量保证 (Engineering & Verification)

### 5.1 零外部依赖技术选型
* **HTTP 请求与缓存**：使用 Node.js 原生 `fetch()`，配合统一的 `RequestCache` 模块。所有发出的网络请求必须按 URL SHA-256 哈希完整缓存响应正文与头部至 `.cache/` 目录。二次执行时优先读本地缓存，断网状态下自动切换为 `--offline` 模式。
* **速率与风控控制**：针对 Wayback Machine 与 Wikidata SPARQL 端点，实现内置令牌桶限流器（Tokens Bucket），请求间隔严格控制在 1.5 秒以上，自动响应 HTTP 429 退避重试。
* **YAML 解析**：为 `overrides.yaml` 提供零依赖的极简子集 YAML 解析器（类似 `doubak-site-generator/src/yaml.js`），支持键值对、嵌套字典与数组列表，无需引入 `yaml` npm 包。

### 5.2 确定性与断网契约测试 (Determinism & Offline Verification)

必须编写以下回归测试用例，纳入 `npm test`（`node:test`）：

1. **`benchmark-37364867.test.js`**：
   * 输入包含墓碑条目 `game/37364867` 的 canonical 切片；
   * 模拟离线环境（网络端点全 Mock 或读取已提交的 Fixture 缓存）；
   * 断言：
     * Enricher 输出的 `subjects.enriched.ndjson` 中标题成功恢复为《情感反诈模拟器》；
     * 别名包含《捞女游戏》与《Revenge on Gold Diggers》；
     * 语言标签正确判定为 `zh-Hans` 与 `en`；
     * Steam AppID 成功对齐为 `3057160`。
2. **`entity-alignment-merge.test.js`**：
   * 构造包含条目 `A`（墓碑，2023 年标记）与条目 `B`（重建，2025 年标记）的双重数据源；
   * 在 `overrides.yaml` 中声明 `A same_as B`；
   * 运行投影计算，断言合并后的作品仅产出 1 个聚合页面，时间线包含 2023 与 2025 两个事件，历史版本计数正确递增。
3. **`zero-network-guarantee.test.js`**：
   * 拦截全局 `globalThis.fetch` 与 `node:https`；
   * 运行带有 `--offline` 标志的 Enricher，断言在完整读取本地缓存的情况下能够 100% 成功生成一致的 NDJSON 文件，未触发任何网络调用。

---

## 6. 实施路线图 (Milestones & Roadmap)

| 阶段 | 交付目标 | 核心工作内容 |
|---|---|---|
| **Phase 1** | **本地侧车与实体声明 (Manual Sidecar & Scaffolding)** | 搭建 `doubak-data-enricher` 基础框架、CLI 入口、`overrides.yaml` 解析器与零网络本地合并机制。立即解决已有数据的紧急手动对齐。 |
| **Phase 2** | **元信息提取器与语言标注 (RawMeta & Lang Detector)** | 实现针对电影/图书/音乐/游戏的 `raw_meta` 启发式规则提取器与 CJK/Latin 零依赖语言判定器，跑通单元测试。 |
| **Phase 3** | **Wayback 快照与外部 ID 反查 (Automated Tombstone Recovery)** | 实现 Wayback Machine CDX API 客户端与 Wikidata SPARQL 客户端；以 `game/37364867` 为标杆跑通全自动墓碑恢复与缓存机制。 |
| **Phase 4** | **垂直领域知识库对接 (Domain Databases)** | 接入 Steam Storefront API 与 TMDB API 检索器，支持高清海报与官方元数据本地固化缓存。 |
| **Phase 5** | **下游流水线贯通 (Downstream Integration & E2E)** | 升级 `doubak-site-generator` 与 `doubak-export-adapters`，支持 `--enrichment` 参数，完成全量真实归档的站点构建与 NeoDB 导出回测。 |

---

## 7. 结语

`doubak-data-enricher` 不仅是一个技术修补工具，更是豆备抵抗数字遗忘（Digital Decay）与平台审查的关键屏障。通过将**客观历史观测**与**外部知识库推断**严谨分离，我们既捍卫了档案的法律取证级真实性，又为每一个个体留住了那些本已被平台宣判“不存在”的文化记忆与人生轨迹。
